---
name: Managing Side Effects Functionally
description: Master functional approaches to side effects including IO types, effect isolation, idempotent operations, and quarantining impure code - essential patterns for reliable TypeScript applications
version: 1.0.0
author: Claude
tags:
  - functional-programming
  - typescript
  - javascript
  - side-effects
  - io-monad
  - purity
  - dependency-injection
  - idempotence
  - fp-ts
---

# Managing Side Effects Functionally

This skill covers functional programming techniques for handling side effects. Side effects are unavoidable in real programs - they're how we interact with the world. The goal isn't to eliminate them, but to control, isolate, and make them predictable.

## Why Side Effect Management Matters

Uncontrolled side effects cause:
- **Unpredictable behavior**: Same function produces different results
- **Testing nightmares**: Must mock globals, databases, time, random
- **Race conditions**: Async operations interfere with each other
- **Hidden dependencies**: Code behavior depends on invisible state
- **Debugging difficulty**: Can't reproduce issues reliably

Functional effect management provides:
- **Predictable programs**: Effects happen when and where you expect
- **Easy testing**: Pure core with thin impure shell
- **Composable operations**: Build complex effects from simple pieces
- **Explicit dependencies**: No hidden state or implicit coupling

---

## 1. What Are Side Effects?

A function has a side effect if it does anything observable besides returning a value. Side effects come in two forms:

### Side Inputs (Causes)

When a function reads from something other than its parameters:

```typescript
// Side input: Reads global state
let globalConfig = { apiUrl: 'https://api.example.com' }
const getApiUrl = (): string => globalConfig.apiUrl
// Same call, different results if globalConfig changes

// Side input: Reads current time
const isExpired = (expiryDate: Date): boolean =>
  expiryDate < new Date()
// Same expiryDate, different results at different times

// Side input: Reads random value
const generateId = (): string =>
  `id-${Math.random().toString(36).slice(2)}`
// Different result every call

// Side input: Reads from DOM
const getInputValue = (id: string): string =>
  (document.getElementById(id) as HTMLInputElement)?.value ?? ''
// Depends on DOM state, not just parameters

// Side input: Reads environment variable
const getDatabaseUrl = (): string =>
  process.env.DATABASE_URL ?? 'localhost:5432'
// Depends on process environment
```

### Side Outputs (Effects)

When a function causes observable changes outside its scope:

```typescript
// Side output: Writes to console
const logUser = (user: User): void => {
  console.log(`User: ${user.name}`)  // Observable effect
}

// Side output: Mutates parameter
const addItem = (arr: string[], item: string): void => {
  arr.push(item)  // Caller's array is modified
}

// Side output: Writes to database
const saveUser = async (user: User): Promise<void> => {
  await database.insert('users', user)  // Persists data
}

// Side output: Sends HTTP request
const sendAnalytics = async (event: AnalyticsEvent): Promise<void> => {
  await fetch('/api/analytics', {
    method: 'POST',
    body: JSON.stringify(event),
  })
}

// Side output: Modifies global state
let requestCount = 0
const trackRequest = (): void => {
  requestCount += 1  // Global mutation
}

// Side output: Triggers DOM update
const showMessage = (message: string): void => {
  document.getElementById('output')!.textContent = message
}
```

### Why Both Are Problems

```typescript
// Side input problem: Unpredictable behavior
const config = { multiplier: 2 }
const calculate = (x: number): number => x * config.multiplier

calculate(5)  // 10
config.multiplier = 3
calculate(5)  // 15 - same input, different output!

// Side output problem: Action at a distance
const items: string[] = ['a', 'b']
const process = (arr: string[]) => {
  arr.push('c')  // Mutates the input
  return arr.length
}

process(items)  // 3
console.log(items)  // ['a', 'b', 'c'] - original modified!

// Combined problem: Hidden coupling
let cache: Record<string, User> = {}

const getUser = async (id: string): Promise<User> => {
  if (cache[id]) return cache[id]  // Side input: reads cache
  const user = await fetchUser(id)
  cache[id] = user  // Side output: writes cache
  return user
}
// Behavior depends on hidden state that might change
```

### The Referential Transparency Test

A function is pure (side-effect free) if you can replace any call with its result without changing program behavior:

```typescript
// PURE: Can substitute
const double = (x: number): number => x * 2
const result = double(5) + double(5)
// Can replace with: 10 + 10 = 20 ✓

// IMPURE: Cannot substitute
let counter = 0
const increment = (): number => {
  counter += 1
  return counter
}
const result = increment() + increment()
// Cannot replace with: 1 + 1 = 2
// Actual result is 1 + 2 = 3 ✗
```

---

## 2. Isolating Side Effects at Program Boundaries

The functional architecture pattern: **Pure Core, Impure Shell**

### The Pattern

```
┌─────────────────────────────────────────┐
│            Impure Shell                 │
│  ┌───────────────────────────────────┐  │
│  │          Pure Core                │  │
│  │                                   │  │
│  │  - Business logic                 │  │
│  │  - Data transformations           │  │
│  │  - Validation                     │  │
│  │  - All decisions                  │  │
│  │                                   │  │
│  └───────────────────────────────────┘  │
│                                         │
│  - HTTP requests                        │
│  - Database operations                  │
│  - File system                          │
│  - User input/output                    │
│  - Logging                              │
│  - Time/randomness                      │
└─────────────────────────────────────────┘
```

### Before: Mixed Pure and Impure

```typescript
// BAD: Business logic mixed with effects
const processOrder = async (orderId: string): Promise<void> => {
  // Impure: Database read
  const order = await database.findOrder(orderId)

  // Impure: Logging
  console.log(`Processing order ${orderId}`)

  // Pure: Business logic (hidden in impure function)
  const discount = order.items.length > 5 ? 0.1 : 0
  const subtotal = order.items.reduce((sum, item) => sum + item.price, 0)
  const total = subtotal * (1 - discount)

  // Impure: Current time
  const processedAt = new Date()

  // Impure: Database write
  await database.updateOrder(orderId, { total, processedAt, status: 'processed' })

  // Impure: Send email
  await emailService.send(order.customerEmail, `Your order total: $${total}`)
}

// Testing requires mocking database, email, console, Date...
```

### After: Pure Core, Impure Shell

```typescript
// PURE CORE: All business logic, no effects
interface OrderItem {
  price: number
  quantity: number
}

interface Order {
  id: string
  customerEmail: string
  items: OrderItem[]
}

interface ProcessedOrder {
  orderId: string
  subtotal: number
  discount: number
  total: number
  processedAt: Date
}

// Pure: Calculate discount based on item count
const calculateDiscount = (itemCount: number): number =>
  itemCount > 5 ? 0.1 : 0

// Pure: Calculate order totals
const calculateOrderTotals = (
  order: Order,
  now: Date
): ProcessedOrder => {
  const subtotal = order.items.reduce(
    (sum, item) => sum + item.price * item.quantity,
    0
  )
  const discount = calculateDiscount(order.items.length)
  const total = subtotal * (1 - discount)

  return {
    orderId: order.id,
    subtotal,
    discount,
    total,
    processedAt: now,
  }
}

// Pure: Format email content
const formatOrderEmail = (processed: ProcessedOrder): string =>
  `Your order total: $${processed.total.toFixed(2)}`

// IMPURE SHELL: Thin layer that coordinates effects
const processOrder = async (orderId: string): Promise<void> => {
  // Effect: Read
  const order = await database.findOrder(orderId)

  // Pure: All business logic
  const processed = calculateOrderTotals(order, new Date())
  const emailContent = formatOrderEmail(processed)

  // Effects: Write
  await database.updateOrder(orderId, {
    total: processed.total,
    processedAt: processed.processedAt,
    status: 'processed',
  })
  await emailService.send(order.customerEmail, emailContent)
  console.log(`Processed order ${orderId}`)
}

// Testing the pure core is trivial:
describe('calculateOrderTotals', () => {
  it('applies 10% discount for orders with more than 5 items', () => {
    const order: Order = {
      id: '123',
      customerEmail: 'test@example.com',
      items: Array(6).fill({ price: 10, quantity: 1 }),
    }
    const now = new Date('2024-01-01')

    const result = calculateOrderTotals(order, now)

    expect(result.discount).toBe(0.1)
    expect(result.total).toBe(54)  // 60 - 10%
  })
})
```

### Program Entry Points as Impure Shell

```typescript
// main.ts - The impure shell at program entry
import { pipe } from 'fp-ts/function'
import * as TE from 'fp-ts/TaskEither'

// Pure business logic
import { validateConfig, processData, formatOutput } from './core'

// Impure adapters
import { readConfigFile, fetchInputData, writeOutput, logError } from './adapters'

// Main: Impure shell that wires everything together
const main = async (): Promise<void> => {
  const result = await pipe(
    // Effect: Read config
    readConfigFile('./config.json'),
    // Pure: Validate
    TE.chainEitherK(validateConfig),
    // Effect: Fetch data
    TE.chain(config => fetchInputData(config.dataUrl)),
    // Pure: Process
    TE.map(processData),
    // Pure: Format
    TE.map(formatOutput),
    // Effect: Write output
    TE.chain(writeOutput),
  )()

  // Effect: Handle result
  if (result._tag === 'Left') {
    logError(result.left)
    process.exit(1)
  }
  console.log('Done!')
}

main()
```

### React: Pure Components, Effects at Boundaries

```typescript
// PURE: Presentational component (no hooks, no effects)
interface UserCardProps {
  user: User
  onEdit: (id: string) => void
  onDelete: (id: string) => void
}

const UserCard: React.FC<UserCardProps> = ({ user, onEdit, onDelete }) => (
  <div className="user-card">
    <h3>{user.name}</h3>
    <p>{user.email}</p>
    <button onClick={() => onEdit(user.id)}>Edit</button>
    <button onClick={() => onDelete(user.id)}>Delete</button>
  </div>
)

// IMPURE SHELL: Container component handles effects
const UserCardContainer: React.FC<{ userId: string }> = ({ userId }) => {
  const [user, setUser] = useState<User | null>(null)
  const [loading, setLoading] = useState(true)

  // Effect: Fetch data
  useEffect(() => {
    fetchUser(userId).then(setUser).finally(() => setLoading(false))
  }, [userId])

  // Effect handlers
  const handleEdit = useCallback((id: string) => {
    navigate(`/users/${id}/edit`)
  }, [])

  const handleDelete = useCallback(async (id: string) => {
    await deleteUser(id)
    navigate('/users')
  }, [])

  if (loading) return <Spinner />
  if (!user) return <NotFound />

  // Render pure component with all data
  return <UserCard user={user} onEdit={handleEdit} onDelete={handleDelete} />
}
```

---

## 3. Idempotent Operations for Async Safety

An operation is **idempotent** if performing it multiple times has the same effect as performing it once. This is crucial for async safety.

### Why Idempotence Matters

```typescript
// NON-IDEMPOTENT: Dangerous in async contexts
const incrementCounter = async (): Promise<void> => {
  const current = await database.get('counter')
  await database.set('counter', current + 1)
}

// Race condition: Two concurrent calls
// Call 1: reads 5
// Call 2: reads 5
// Call 1: writes 6
// Call 2: writes 6
// Expected: 7, Actual: 6

// IDEMPOTENT: Safe for retries and concurrency
const setCounter = async (value: number): Promise<void> => {
  await database.set('counter', value)
}
// Multiple calls with same value = same result
```

### Patterns for Idempotence

#### 1. Use Idempotency Keys

```typescript
interface PaymentRequest {
  idempotencyKey: string  // Client-generated unique ID
  amount: number
  currency: string
  customerId: string
}

const processPayment = async (request: PaymentRequest): Promise<PaymentResult> => {
  // Check if we've already processed this exact request
  const existing = await database.findPayment(request.idempotencyKey)
  if (existing) {
    return existing  // Return cached result, don't process again
  }

  // Process the payment
  const result = await paymentGateway.charge(request)

  // Store with idempotency key for future lookups
  await database.savePayment(request.idempotencyKey, result)

  return result
}

// Client code:
const pay = async () => {
  const request: PaymentRequest = {
    idempotencyKey: `pay-${orderId}-${Date.now()}`,  // Unique per attempt
    amount: 99.99,
    currency: 'USD',
    customerId: user.id,
  }

  // Safe to retry on network failure
  return await retryWithBackoff(() => processPayment(request))
}
```

#### 2. Use Conditional Updates

```typescript
// NON-IDEMPOTENT: Increment
const addToBalance = async (userId: string, amount: number): Promise<void> => {
  await database.query(
    'UPDATE accounts SET balance = balance + $1 WHERE user_id = $2',
    [amount, userId]
  )
}

// IDEMPOTENT: Set to specific value with version check
interface BalanceUpdate {
  userId: string
  newBalance: number
  expectedVersion: number
}

const updateBalance = async (update: BalanceUpdate): Promise<boolean> => {
  const result = await database.query(
    `UPDATE accounts
     SET balance = $1, version = version + 1
     WHERE user_id = $2 AND version = $3`,
    [update.newBalance, update.userId, update.expectedVersion]
  )
  return result.rowCount > 0  // False if version mismatch (concurrent update)
}

// Usage with optimistic locking
const transferFunds = async (
  fromId: string,
  toId: string,
  amount: number
): Promise<Either<TransferError, void>> => {
  const [from, to] = await Promise.all([
    database.getAccount(fromId),
    database.getAccount(toId),
  ])

  if (from.balance < amount) {
    return E.left({ type: 'InsufficientFunds' })
  }

  // Idempotent updates with version checks
  const fromUpdated = await updateBalance({
    userId: fromId,
    newBalance: from.balance - amount,
    expectedVersion: from.version,
  })

  if (!fromUpdated) {
    return E.left({ type: 'ConcurrentModification', account: 'source' })
  }

  const toUpdated = await updateBalance({
    userId: toId,
    newBalance: to.balance + amount,
    expectedVersion: to.version,
  })

  if (!toUpdated) {
    // Rollback source account
    await updateBalance({
      userId: fromId,
      newBalance: from.balance,
      expectedVersion: from.version + 1,
    })
    return E.left({ type: 'ConcurrentModification', account: 'destination' })
  }

  return E.right(undefined)
}
```

#### 3. Use PUT Semantics Over POST

```typescript
// NON-IDEMPOTENT: POST creates new resource each time
// POST /api/orders
const createOrder = async (data: OrderData): Promise<Order> => {
  const id = generateId()  // New ID each call
  return await database.insert({ id, ...data })
}

// IDEMPOTENT: PUT replaces resource at specific ID
// PUT /api/orders/:id
const upsertOrder = async (id: string, data: OrderData): Promise<Order> => {
  return await database.upsert({ id, ...data })
}
// Multiple calls with same ID and data = same result

// Client generates ID:
const placeOrder = async (items: CartItem[]): Promise<Order> => {
  const orderId = `order-${userId}-${Date.now()}`  // Deterministic ID
  return await api.put(`/orders/${orderId}`, { items })
  // Safe to retry - same order won't be created twice
}
```

#### 4. Design for "At Least Once" Delivery

```typescript
// Message handlers should be idempotent
interface OrderMessage {
  messageId: string  // Unique message identifier
  orderId: string
  action: 'process' | 'ship' | 'cancel'
}

const handleOrderMessage = async (message: OrderMessage): Promise<void> => {
  // Track processed messages
  const alreadyProcessed = await messageStore.exists(message.messageId)
  if (alreadyProcessed) {
    console.log(`Message ${message.messageId} already processed, skipping`)
    return
  }

  // Process the message
  switch (message.action) {
    case 'process':
      await processOrder(message.orderId)
      break
    case 'ship':
      await shipOrder(message.orderId)
      break
    case 'cancel':
      await cancelOrder(message.orderId)
      break
  }

  // Mark as processed
  await messageStore.markProcessed(message.messageId)
}

// Even better: Make the operations themselves idempotent
const shipOrder = async (orderId: string): Promise<void> => {
  const order = await database.getOrder(orderId)

  // Idempotent check: Already shipped? Do nothing
  if (order.status === 'shipped') {
    return
  }

  // Only ship if in correct state
  if (order.status !== 'processed') {
    throw new InvalidStateError(`Cannot ship order in ${order.status} state`)
  }

  await shippingService.createShipment(order)
  await database.updateOrder(orderId, { status: 'shipped' })
}
```

### Race Condition Mitigation

```typescript
// Pattern: Atomic read-modify-write with transactions
const safeIncrement = async (key: string): Promise<number> => {
  return await database.transaction(async (tx) => {
    const current = await tx.get(key)
    const newValue = (current ?? 0) + 1
    await tx.set(key, newValue)
    return newValue
  })
}

// Pattern: Compare-and-swap (CAS)
const compareAndSwap = async <T>(
  key: string,
  expectedValue: T,
  newValue: T
): Promise<boolean> => {
  const result = await redis.eval(`
    if redis.call('get', KEYS[1]) == ARGV[1] then
      redis.call('set', KEYS[1], ARGV[2])
      return 1
    else
      return 0
    end
  `, 1, key, JSON.stringify(expectedValue), JSON.stringify(newValue))
  return result === 1
}

// Pattern: Distributed locks for non-idempotent operations
import { Mutex } from 'async-mutex'

const orderMutexes = new Map<string, Mutex>()

const getMutex = (orderId: string): Mutex => {
  if (!orderMutexes.has(orderId)) {
    orderMutexes.set(orderId, new Mutex())
  }
  return orderMutexes.get(orderId)!
}

const processOrderSafely = async (orderId: string): Promise<void> => {
  const mutex = getMutex(orderId)

  await mutex.runExclusive(async () => {
    // Only one execution at a time per orderId
    await processOrder(orderId)
  })
}
```

---

## 4. IO Type for Synchronous Effects

The `IO` type represents a synchronous computation that may have side effects. It's a function that takes no arguments and returns a value.

### What is IO?

```typescript
// IO is just a thunk - a function waiting to be called
type IO<A> = () => A

// Creating IO values (doesn't execute the effect)
const getCurrentTime: IO<Date> = () => new Date()
const getRandomNumber: IO<number> = () => Math.random()
const readEnvVar = (name: string): IO<string | undefined> =>
  () => process.env[name]

// The effect only happens when you call the function
const time1 = getCurrentTime()  // Effect happens now
const time2 = getCurrentTime()  // Effect happens again
```

### Why Wrap Effects in IO?

```typescript
// WITHOUT IO: Effect happens immediately, can't compose
const logAndReturn = <A>(a: A): A => {
  console.log(a)  // Effect happens during function creation
  return a
}

// WITH IO: Effect is deferred, can be composed
const logAndReturn = <A>(a: A): IO<A> => () => {
  console.log(a)
  return a
}

// Composing IO operations
import { pipe } from 'fp-ts/function'
import * as IO from 'fp-ts/IO'

const program: IO<void> = pipe(
  getCurrentTime,
  IO.map(date => date.toISOString()),
  IO.chain(iso => () => console.log(`Current time: ${iso}`))
)

// Nothing has happened yet! Execute when ready:
program()  // Now effects happen
```

### fp-ts IO Module

```typescript
import { pipe } from 'fp-ts/function'
import * as IO from 'fp-ts/IO'

// IO.of: Lift a pure value into IO
const pureValue: IO.IO<number> = IO.of(42)

// IO.map: Transform the value inside IO
const doubled: IO.IO<number> = pipe(
  IO.of(21),
  IO.map(n => n * 2)
)

// IO.chain: Sequence IO operations (flatMap)
const getAndLog: IO.IO<void> = pipe(
  () => new Date(),
  IO.chain(date => () => console.log(date.toISOString()))
)

// IO.chainFirst: Run an effect but keep the original value
const loggedValue: IO.IO<number> = pipe(
  IO.of(42),
  IO.chainFirst(n => () => console.log(`Value is: ${n}`)),
  IO.map(n => n + 1)  // Still has access to 42, not console.log result
)

// Combining multiple IOs
const combined: IO.IO<string> = pipe(
  IO.Do,
  IO.bind('time', () => () => new Date()),
  IO.bind('random', () => () => Math.random()),
  IO.map(({ time, random }) => `${time.toISOString()}: ${random}`)
)
```

### Practical IO Examples

```typescript
import * as IO from 'fp-ts/IO'
import { pipe } from 'fp-ts/function'

// Console operations as IO
const log = (message: string): IO.IO<void> =>
  () => console.log(message)

const warn = (message: string): IO.IO<void> =>
  () => console.warn(message)

const error = (message: string): IO.IO<void> =>
  () => console.error(message)

// Reading from environment
const getEnv = (key: string): IO.IO<string | undefined> =>
  () => process.env[key]

const requireEnv = (key: string): IO.IO<string> =>
  () => {
    const value = process.env[key]
    if (!value) throw new Error(`Missing env var: ${key}`)
    return value
  }

// DOM operations as IO
const getElementById = (id: string): IO.IO<HTMLElement | null> =>
  () => document.getElementById(id)

const setTextContent = (element: HTMLElement, text: string): IO.IO<void> =>
  () => { element.textContent = text }

// Random and time
const random: IO.IO<number> = () => Math.random()
const now: IO.IO<Date> = () => new Date()

// Building a program
const displayCurrentTime = (elementId: string): IO.IO<void> =>
  pipe(
    IO.Do,
    IO.bind('element', () => getElementById(elementId)),
    IO.bind('time', () => now),
    IO.chain(({ element, time }) =>
      element
        ? setTextContent(element, time.toLocaleTimeString())
        : log(`Element ${elementId} not found`)
    )
  )

// Program is just data - execute when ready
const program = displayCurrentTime('clock')
program()  // Actually runs the effects
```

### IO vs Immediate Effects

```typescript
// IMMEDIATE: Effects happen during setup
class Logger {
  constructor() {
    console.log('Logger initialized')  // Effect in constructor!
  }

  log(msg: string): void {
    console.log(msg)
  }
}

// Just creating the class causes effects
const logger = new Logger()  // "Logger initialized" printed

// DEFERRED with IO: Effects are controlled
const createLogger = (name: string): IO.IO<Logger> => () => {
  console.log(`Logger ${name} initialized`)
  return {
    log: (msg: string) => console.log(`[${name}] ${msg}`),
  }
}

// Nothing printed yet
const loggerProgram = createLogger('app')
// Now it runs
const logger = loggerProgram()  // "Logger app initialized"
```

---

## 5. Quarantining Impure Code

Quarantining means isolating impure code into specific modules that are clearly marked as "effectful", keeping the rest of your codebase pure.

### Module Structure for Quarantine

```
src/
├── core/                    # PURE: Business logic
│   ├── domain/
│   │   ├── user.ts          # User type and pure functions
│   │   ├── order.ts         # Order type and calculations
│   │   └── validation.ts    # Pure validation functions
│   └── services/
│       ├── pricing.ts       # Pure pricing calculations
│       └── discount.ts      # Pure discount rules
│
├── adapters/                # IMPURE: External world
│   ├── database/
│   │   ├── userRepo.ts      # Database operations
│   │   └── orderRepo.ts
│   ├── http/
│   │   ├── userApi.ts       # HTTP clients
│   │   └── paymentApi.ts
│   ├── logging/
│   │   └── logger.ts        # Console/file logging
│   └── config/
│       └── env.ts           # Environment variables
│
├── effects/                 # Effect type definitions
│   ├── io.ts               # IO effect utilities
│   └── task.ts             # Async effect utilities
│
└── main.ts                  # IMPURE: Entry point, wires everything
```

### Pure Domain Layer

```typescript
// core/domain/order.ts - PURE

export interface OrderItem {
  productId: string
  name: string
  price: number
  quantity: number
}

export interface Order {
  id: string
  customerId: string
  items: readonly OrderItem[]
  status: OrderStatus
  createdAt: Date
}

export type OrderStatus = 'pending' | 'confirmed' | 'shipped' | 'delivered'

// Pure functions - no effects
export const calculateSubtotal = (items: readonly OrderItem[]): number =>
  items.reduce((sum, item) => sum + item.price * item.quantity, 0)

export const calculateTax = (subtotal: number, taxRate: number): number =>
  subtotal * taxRate

export const calculateTotal = (
  subtotal: number,
  tax: number,
  discount: number
): number =>
  Math.max(0, subtotal + tax - discount)

export const canShip = (order: Order): boolean =>
  order.status === 'confirmed' && order.items.length > 0

export const canCancel = (order: Order): boolean =>
  order.status === 'pending' || order.status === 'confirmed'

// State transitions as pure functions
export const confirmOrder = (order: Order): Order => ({
  ...order,
  status: 'confirmed',
})

export const shipOrder = (order: Order): Order => ({
  ...order,
  status: 'shipped',
})
```

### Impure Adapter Layer

```typescript
// adapters/database/orderRepo.ts - IMPURE (clearly marked)

import { Order, OrderStatus } from '../../core/domain/order'
import * as TE from 'fp-ts/TaskEither'
import { pipe } from 'fp-ts/function'

// Mark as impure in the module documentation
/**
 * @module OrderRepository
 * @impure This module performs database operations
 */

export interface OrderRepository {
  findById: (id: string) => TE.TaskEither<DatabaseError, Order>
  findByCustomer: (customerId: string) => TE.TaskEither<DatabaseError, Order[]>
  save: (order: Order) => TE.TaskEither<DatabaseError, Order>
  updateStatus: (id: string, status: OrderStatus) => TE.TaskEither<DatabaseError, void>
}

export type DatabaseError =
  | { type: 'NotFound'; id: string }
  | { type: 'ConnectionError'; message: string }
  | { type: 'QueryError'; message: string }

// Implementation with effects
export const createOrderRepository = (
  pool: DatabasePool
): OrderRepository => ({
  findById: (id) =>
    TE.tryCatch(
      async () => {
        const result = await pool.query('SELECT * FROM orders WHERE id = $1', [id])
        if (result.rows.length === 0) {
          throw { type: 'NotFound', id }
        }
        return mapRowToOrder(result.rows[0])
      },
      (error): DatabaseError => {
        if ((error as any).type === 'NotFound') return error as DatabaseError
        return { type: 'QueryError', message: String(error) }
      }
    ),

  findByCustomer: (customerId) =>
    TE.tryCatch(
      async () => {
        const result = await pool.query(
          'SELECT * FROM orders WHERE customer_id = $1',
          [customerId]
        )
        return result.rows.map(mapRowToOrder)
      },
      (error): DatabaseError => ({
        type: 'QueryError',
        message: String(error),
      })
    ),

  save: (order) =>
    TE.tryCatch(
      async () => {
        await pool.query(
          `INSERT INTO orders (id, customer_id, items, status, created_at)
           VALUES ($1, $2, $3, $4, $5)
           ON CONFLICT (id) DO UPDATE SET
           items = $3, status = $4`,
          [order.id, order.customerId, JSON.stringify(order.items),
           order.status, order.createdAt]
        )
        return order
      },
      (error): DatabaseError => ({
        type: 'QueryError',
        message: String(error),
      })
    ),

  updateStatus: (id, status) =>
    TE.tryCatch(
      async () => {
        await pool.query(
          'UPDATE orders SET status = $1 WHERE id = $2',
          [status, id]
        )
      },
      (error): DatabaseError => ({
        type: 'QueryError',
        message: String(error),
      })
    ),
})
```

### Wiring Pure and Impure at Boundaries

```typescript
// application/orderService.ts - Combines pure logic with impure operations

import { pipe } from 'fp-ts/function'
import * as TE from 'fp-ts/TaskEither'
import * as E from 'fp-ts/Either'
import * as Order from '../core/domain/order'
import { OrderRepository } from '../adapters/database/orderRepo'
import { PaymentGateway } from '../adapters/http/paymentApi'
import { Logger } from '../adapters/logging/logger'

export interface OrderService {
  processOrder: (orderId: string) => TE.TaskEither<ProcessError, Order.Order>
}

export type ProcessError =
  | { type: 'NotFound'; orderId: string }
  | { type: 'InvalidState'; message: string }
  | { type: 'PaymentFailed'; reason: string }
  | { type: 'DatabaseError'; message: string }

export const createOrderService = (
  orderRepo: OrderRepository,
  paymentGateway: PaymentGateway,
  logger: Logger
): OrderService => ({
  processOrder: (orderId) =>
    pipe(
      // Impure: Log start
      TE.fromIO(logger.info(`Processing order ${orderId}`)),

      // Impure: Fetch order
      TE.chain(() => pipe(
        orderRepo.findById(orderId),
        TE.mapLeft((e): ProcessError =>
          e.type === 'NotFound'
            ? { type: 'NotFound', orderId }
            : { type: 'DatabaseError', message: e.message }
        )
      )),

      // Pure: Validate state
      TE.chainEitherK((order) =>
        Order.canShip(order)
          ? E.right(order)
          : E.left<ProcessError>({
              type: 'InvalidState',
              message: `Cannot process order in ${order.status} state`
            })
      ),

      // Pure: Calculate total
      TE.map((order) => {
        const subtotal = Order.calculateSubtotal(order.items)
        const tax = Order.calculateTax(subtotal, 0.08)
        const total = Order.calculateTotal(subtotal, tax, 0)
        return { order, total }
      }),

      // Impure: Process payment
      TE.chain(({ order, total }) =>
        pipe(
          paymentGateway.charge(order.customerId, total),
          TE.mapLeft((e): ProcessError => ({
            type: 'PaymentFailed',
            reason: e.message
          })),
          TE.map(() => order)
        )
      ),

      // Pure: Update state
      TE.map(Order.shipOrder),

      // Impure: Save updated order
      TE.chain((order) =>
        pipe(
          orderRepo.save(order),
          TE.mapLeft((e): ProcessError => ({
            type: 'DatabaseError',
            message: e.message
          }))
        )
      ),

      // Impure: Log success
      TE.chainFirst((order) =>
        TE.fromIO(logger.info(`Order ${order.id} shipped successfully`))
      )
    ),
})
```

---

## 6. Making Impure Functions Pure Through Dependency Injection

Instead of calling impure operations directly, accept them as parameters. This makes functions pure and testable.

### The Problem with Direct Dependencies

```typescript
// IMPURE: Direct dependency on external systems
const processUser = async (userId: string): Promise<ProcessResult> => {
  // Direct call to database
  const user = await database.findUser(userId)

  // Direct call to current time
  const now = new Date()

  // Direct call to random
  const token = generateToken()

  // Direct call to logger
  console.log(`Processing user ${userId}`)

  // Direct call to external API
  const enriched = await enrichmentApi.enrich(user)

  return { user: enriched, processedAt: now, token }
}

// Testing is painful:
// - Must mock database
// - Must mock Date
// - Must mock Math.random
// - Must suppress console.log
// - Must mock external API
```

### Solution: Dependency Injection

```typescript
// PURE(ish): Dependencies are injected
interface ProcessUserDeps {
  findUser: (id: string) => Promise<User>
  getCurrentTime: () => Date
  generateToken: () => string
  log: (message: string) => void
  enrichUser: (user: User) => Promise<EnrichedUser>
}

const processUser = (deps: ProcessUserDeps) =>
  async (userId: string): Promise<ProcessResult> => {
    const user = await deps.findUser(userId)
    const now = deps.getCurrentTime()
    const token = deps.generateToken()
    deps.log(`Processing user ${userId}`)
    const enriched = await deps.enrichUser(user)
    return { user: enriched, processedAt: now, token }
  }

// Production usage:
const productionDeps: ProcessUserDeps = {
  findUser: database.findUser,
  getCurrentTime: () => new Date(),
  generateToken: () => crypto.randomUUID(),
  log: console.log,
  enrichUser: enrichmentApi.enrich,
}

const processUserProd = processUser(productionDeps)
await processUserProd('user-123')

// Testing is trivial:
describe('processUser', () => {
  it('enriches user and returns result', async () => {
    const mockUser: User = { id: '123', name: 'Test' }
    const mockEnriched: EnrichedUser = { ...mockUser, score: 100 }
    const fixedTime = new Date('2024-01-01')

    const testDeps: ProcessUserDeps = {
      findUser: async () => mockUser,
      getCurrentTime: () => fixedTime,
      generateToken: () => 'test-token',
      log: () => {},  // No-op
      enrichUser: async () => mockEnriched,
    }

    const result = await processUser(testDeps)('123')

    expect(result).toEqual({
      user: mockEnriched,
      processedAt: fixedTime,
      token: 'test-token',
    })
  })
})
```

### Reader Pattern for Dependency Injection

```typescript
import { pipe } from 'fp-ts/function'
import * as R from 'fp-ts/Reader'
import * as RT from 'fp-ts/ReaderTask'
import * as RTE from 'fp-ts/ReaderTaskEither'

// Define environment (dependencies)
interface AppEnv {
  userRepo: UserRepository
  logger: Logger
  config: AppConfig
  clock: { now: () => Date }
}

// Functions that read from environment
const findUser = (id: string): RTE.ReaderTaskEither<AppEnv, Error, User> =>
  (env) => env.userRepo.findById(id)

const logMessage = (msg: string): RT.ReaderTask<AppEnv, void> =>
  (env) => async () => env.logger.info(msg)

const getConfig = <K extends keyof AppConfig>(key: K): R.Reader<AppEnv, AppConfig[K]> =>
  (env) => env.config[key]

const getCurrentTime: R.Reader<AppEnv, Date> =
  (env) => env.clock.now()

// Compose operations that need environment
const processUserWithReader = (
  userId: string
): RTE.ReaderTaskEither<AppEnv, ProcessError, ProcessResult> =>
  pipe(
    // Log start
    RTE.fromReaderTask(logMessage(`Processing user ${userId}`)),
    // Fetch user
    RTE.chain(() => findUser(userId)),
    // Get current time
    RTE.bindTo('user'),
    RTE.bind('time', () => RTE.fromReader(getCurrentTime)),
    // Build result
    RTE.map(({ user, time }) => ({
      user,
      processedAt: time,
    }))
  )

// Run with production environment
const productionEnv: AppEnv = {
  userRepo: createUserRepository(databasePool),
  logger: createLogger('app'),
  config: loadConfig(),
  clock: { now: () => new Date() },
}

const result = await processUserWithReader('user-123')(productionEnv)()

// Run with test environment
const testEnv: AppEnv = {
  userRepo: {
    findById: () => TE.right(mockUser),
  } as UserRepository,
  logger: { info: () => {} } as Logger,
  config: { apiUrl: 'http://test' } as AppConfig,
  clock: { now: () => new Date('2024-01-01') },
}

const testResult = await processUserWithReader('user-123')(testEnv)()
```

### Functional Core with Injected Effects

```typescript
// Pure core that accepts effect functions
interface OrderProcessingEffects {
  // Queries (read effects)
  getOrder: (id: string) => TE.TaskEither<Error, Order>
  getInventory: (productId: string) => TE.TaskEither<Error, number>

  // Commands (write effects)
  saveOrder: (order: Order) => TE.TaskEither<Error, void>
  sendNotification: (email: string, message: string) => TE.TaskEither<Error, void>

  // Environment
  getCurrentTime: () => Date
}

// Pure business logic - just transforms data
const calculateShippingDate = (orderDate: Date, priority: Priority): Date => {
  const days = priority === 'express' ? 1 : 5
  const shipping = new Date(orderDate)
  shipping.setDate(shipping.getDate() + days)
  return shipping
}

const canFulfill = (order: Order, inventory: Map<string, number>): boolean =>
  order.items.every(item =>
    (inventory.get(item.productId) ?? 0) >= item.quantity
  )

// Orchestration layer using injected effects
const processOrderWorkflow = (effects: OrderProcessingEffects) =>
  (orderId: string): TE.TaskEither<ProcessError, ProcessedOrder> =>
    pipe(
      // Fetch data (effects)
      effects.getOrder(orderId),
      TE.bindTo('order'),
      TE.bind('inventory', ({ order }) =>
        pipe(
          order.items.map(item =>
            pipe(
              effects.getInventory(item.productId),
              TE.map(qty => [item.productId, qty] as const)
            )
          ),
          TE.sequenceArray,
          TE.map(entries => new Map(entries))
        )
      ),

      // Pure validation
      TE.chainEitherK(({ order, inventory }) =>
        canFulfill(order, inventory)
          ? E.right({ order, inventory })
          : E.left({ type: 'OutOfStock' as const })
      ),

      // Pure calculation
      TE.map(({ order }) => {
        const now = effects.getCurrentTime()
        const shippingDate = calculateShippingDate(now, order.priority)
        return {
          ...order,
          status: 'confirmed' as const,
          confirmedAt: now,
          estimatedShipping: shippingDate,
        }
      }),

      // Write effects
      TE.chainFirst(processedOrder => effects.saveOrder(processedOrder)),
      TE.chainFirst(processedOrder =>
        effects.sendNotification(
          processedOrder.customerEmail,
          `Order confirmed! Ships by ${processedOrder.estimatedShipping.toDateString()}`
        )
      )
    )
```

---

## 7. When Side Effects Are Acceptable

Side effects aren't always bad. Here's guidance on when they're acceptable.

### Logging and Telemetry

```typescript
// ACCEPTABLE: Observability side effects
// These don't affect business logic correctness

const processPayment = async (payment: Payment): Promise<PaymentResult> => {
  // OK: Logging for observability
  logger.info('Processing payment', { paymentId: payment.id })

  const startTime = performance.now()

  const result = await paymentGateway.process(payment)

  // OK: Metrics for monitoring
  metrics.recordTiming('payment.processing', performance.now() - startTime)
  metrics.increment('payment.processed', { status: result.status })

  // OK: Tracing for debugging
  span.setTag('payment.status', result.status)

  return result
}

// WHY IT'S OK:
// - Doesn't affect return value
// - Failure doesn't break business logic
// - Can be disabled in tests
// - Aids debugging/monitoring
```

### Caching (When Semantically Transparent)

```typescript
// ACCEPTABLE: Caching that's semantically transparent
// Returns same result, just faster on subsequent calls

const cache = new Map<string, User>()

const getUserCached = async (id: string): Promise<User> => {
  // Cache hit - return immediately
  const cached = cache.get(id)
  if (cached) {
    return cached
  }

  // Cache miss - fetch and store
  const user = await database.findUser(id)
  cache.set(id, user)
  return user
}

// WHY IT'S OK:
// - Same input always returns same output (eventually)
// - Side effect (caching) is an optimization detail
// - No observable behavior change from caller's perspective

// CAUTION: Cache invalidation can introduce subtle bugs
// Consider using established caching libraries
```

### Configuration Loading at Startup

```typescript
// ACCEPTABLE: One-time initialization effects
// Load config once, use immutably thereafter

// config.ts
interface AppConfig {
  readonly apiUrl: string
  readonly maxRetries: number
  readonly featureFlags: readonly string[]
}

// Side effect: Reads environment (but only once at startup)
const loadConfig = (): AppConfig => ({
  apiUrl: process.env.API_URL ?? 'http://localhost:3000',
  maxRetries: parseInt(process.env.MAX_RETRIES ?? '3', 10),
  featureFlags: (process.env.FEATURE_FLAGS ?? '').split(',').filter(Boolean),
})

// Freeze to prevent accidental mutation
export const config: AppConfig = Object.freeze(loadConfig())

// WHY IT'S OK:
// - Happens once at startup
// - Result is immutable
// - Makes config available without passing everywhere
// - Established pattern in most applications
```

### Development/Debug Tools

```typescript
// ACCEPTABLE: Debug-only side effects

const debugLog = (message: string, data?: unknown): void => {
  if (process.env.NODE_ENV === 'development') {
    console.log(`[DEBUG] ${message}`, data)
  }
}

const processData = (input: InputData): OutputData => {
  debugLog('Processing input', input)  // OK in development

  const result = transformData(input)

  debugLog('Transform complete', result)

  return result
}

// WHY IT'S OK:
// - Only affects development experience
// - Completely absent in production
// - Aids debugging
```

### Side Effects to Avoid

```typescript
// AVOID: Side effects that affect correctness

// BAD: Global mutable state affecting logic
let globalDiscount = 0.1
const calculatePrice = (base: number): number => base * (1 - globalDiscount)
// Any code can change globalDiscount, breaking calculatePrice

// BAD: Implicit dependencies
const validateUser = (user: User): boolean => {
  // Reads from somewhere that might change
  const rules = getValidationRules()  // Where do these come from?
  return rules.every(rule => rule(user))
}

// BAD: Non-deterministic business logic
const shouldRetry = (): boolean => Math.random() < 0.5
// Can't test, can't predict behavior

// BAD: Mutation of input arguments
const processItems = (items: Item[]): void => {
  items.forEach(item => {
    item.processed = true  // Mutates caller's data
  })
}
```

---

## Practical Exercises

### Exercise 1: Identify Side Effects

Classify each function as pure or impure. If impure, identify the side effect type.

```typescript
// 1a
const formatDate = (date: Date): string =>
  date.toISOString().split('T')[0]

// 1b
const formatCurrentDate = (): string =>
  new Date().toISOString().split('T')[0]

// 1c
const memoize = <A, B>(fn: (a: A) => B): (a: A) => B => {
  const cache = new Map<A, B>()
  return (a: A) => {
    if (cache.has(a)) return cache.get(a)!
    const result = fn(a)
    cache.set(a, result)
    return result
  }
}

// 1d
const sortUsers = (users: User[]): User[] =>
  users.sort((a, b) => a.name.localeCompare(b.name))

// 1e
const getUserAge = (user: User): number => {
  const today = new Date()
  return today.getFullYear() - user.birthYear
}
```

<details>
<summary>Solutions</summary>

```typescript
// 1a: PURE
// Takes a Date, returns a string
// No side inputs or outputs
const formatDate = (date: Date): string =>
  date.toISOString().split('T')[0]

// 1b: IMPURE (side input)
// Reads current time - different output at different times
const formatCurrentDate = (): string =>
  new Date().toISOString().split('T')[0]

// FIX: Accept date as parameter
const formatCurrentDate = (now: Date): string =>
  now.toISOString().split('T')[0]

// 1c: IMPURE (side output) but semantically transparent
// Mutates internal cache state
// However, from caller's perspective, same input = same output
// This is an acceptable side effect (caching)
const memoize = <A, B>(fn: (a: A) => B): (a: A) => B => {
  const cache = new Map<A, B>()
  return (a: A) => {
    if (cache.has(a)) return cache.get(a)!
    const result = fn(a)
    cache.set(a, result)
    return result
  }
}

// 1d: IMPURE (side output)
// Array.sort() mutates the original array!
const sortUsers = (users: User[]): User[] =>
  users.sort((a, b) => a.name.localeCompare(b.name))

// FIX: Create copy before sorting
const sortUsers = (users: readonly User[]): User[] =>
  [...users].sort((a, b) => a.name.localeCompare(b.name))

// 1e: IMPURE (side input)
// Reads current time
const getUserAge = (user: User): number => {
  const today = new Date()
  return today.getFullYear() - user.birthYear
}

// FIX: Accept current date as parameter
const getUserAge = (user: User, today: Date): number =>
  today.getFullYear() - user.birthYear
```
</details>

### Exercise 2: Isolate Effects

Refactor this impure function into a pure core with an impure shell.

```typescript
const processOrder = async (orderId: string): Promise<void> => {
  console.log(`Starting order processing: ${orderId}`)

  const order = await fetch(`/api/orders/${orderId}`).then(r => r.json())

  if (order.items.length === 0) {
    throw new Error('Empty order')
  }

  const subtotal = order.items.reduce(
    (sum: number, item: { price: number; quantity: number }) =>
      sum + item.price * item.quantity,
    0
  )

  const discount = subtotal > 100 ? 0.1 : 0
  const tax = subtotal * 0.08
  const total = subtotal * (1 - discount) + tax

  const processedOrder = {
    ...order,
    subtotal,
    discount,
    tax,
    total,
    processedAt: new Date().toISOString(),
  }

  await fetch(`/api/orders/${orderId}`, {
    method: 'PUT',
    body: JSON.stringify(processedOrder),
  })

  console.log(`Order processed: ${orderId}, total: $${total}`)
}
```

<details>
<summary>Solution</summary>

```typescript
// PURE CORE: Business logic with no effects

interface OrderItem {
  price: number
  quantity: number
}

interface Order {
  id: string
  items: OrderItem[]
}

interface ProcessedOrder extends Order {
  subtotal: number
  discount: number
  tax: number
  total: number
  processedAt: string
}

// Pure: Validates order
const validateOrder = (order: Order): Either<string, Order> =>
  order.items.length === 0
    ? E.left('Empty order')
    : E.right(order)

// Pure: Calculates subtotal
const calculateSubtotal = (items: readonly OrderItem[]): number =>
  items.reduce((sum, item) => sum + item.price * item.quantity, 0)

// Pure: Determines discount based on subtotal
const calculateDiscount = (subtotal: number): number =>
  subtotal > 100 ? 0.1 : 0

// Pure: Calculates tax
const calculateTax = (subtotal: number): number =>
  subtotal * 0.08

// Pure: Calculates final total
const calculateTotal = (subtotal: number, discount: number, tax: number): number =>
  subtotal * (1 - discount) + tax

// Pure: Transforms order to processed order
const processOrderData = (order: Order, timestamp: string): ProcessedOrder => {
  const subtotal = calculateSubtotal(order.items)
  const discount = calculateDiscount(subtotal)
  const tax = calculateTax(subtotal)
  const total = calculateTotal(subtotal, discount, tax)

  return {
    ...order,
    subtotal,
    discount,
    tax,
    total,
    processedAt: timestamp,
  }
}

// IMPURE SHELL: Orchestrates effects

interface OrderEffects {
  fetchOrder: (id: string) => TE.TaskEither<Error, Order>
  saveOrder: (order: ProcessedOrder) => TE.TaskEither<Error, void>
  log: (message: string) => void
  getCurrentTime: () => Date
}

const processOrder = (effects: OrderEffects) =>
  (orderId: string): TE.TaskEither<Error, ProcessedOrder> =>
    pipe(
      // Effect: Log start
      TE.fromIO(() => effects.log(`Starting order processing: ${orderId}`)),

      // Effect: Fetch order
      TE.chain(() => effects.fetchOrder(orderId)),

      // Pure: Validate
      TE.chainEitherK(order =>
        pipe(
          validateOrder(order),
          E.mapLeft(msg => new Error(msg))
        )
      ),

      // Pure: Process
      TE.map(order => {
        const timestamp = effects.getCurrentTime().toISOString()
        return processOrderData(order, timestamp)
      }),

      // Effect: Save
      TE.chainFirst(processedOrder => effects.saveOrder(processedOrder)),

      // Effect: Log completion
      TE.chainFirst(processedOrder =>
        TE.fromIO(() =>
          effects.log(`Order processed: ${orderId}, total: $${processedOrder.total}`)
        )
      )
    )

// Production wiring
const productionEffects: OrderEffects = {
  fetchOrder: (id) =>
    TE.tryCatch(
      () => fetch(`/api/orders/${id}`).then(r => r.json()),
      (e) => new Error(String(e))
    ),
  saveOrder: (order) =>
    TE.tryCatch(
      () => fetch(`/api/orders/${order.id}`, {
        method: 'PUT',
        body: JSON.stringify(order),
      }).then(() => undefined),
      (e) => new Error(String(e))
    ),
  log: console.log,
  getCurrentTime: () => new Date(),
}

const processOrderProd = processOrder(productionEffects)

// Easy to test!
describe('processOrderData', () => {
  it('applies 10% discount for orders over $100', () => {
    const order: Order = {
      id: '123',
      items: [{ price: 50, quantity: 3 }],  // $150 subtotal
    }

    const result = processOrderData(order, '2024-01-01T00:00:00Z')

    expect(result.subtotal).toBe(150)
    expect(result.discount).toBe(0.1)
    expect(result.tax).toBe(12)  // 150 * 0.08
    expect(result.total).toBe(147)  // 150 * 0.9 + 12
  })
})
```
</details>

### Exercise 3: Make It Idempotent

This payment processor has race condition issues. Make it idempotent.

```typescript
const processPayment = async (
  userId: string,
  amount: number
): Promise<PaymentResult> => {
  const user = await database.getUser(userId)

  if (user.balance < amount) {
    return { success: false, reason: 'Insufficient funds' }
  }

  // Deduct from balance
  await database.updateUser(userId, {
    balance: user.balance - amount,
  })

  // Record transaction
  const transactionId = generateId()
  await database.insertTransaction({
    id: transactionId,
    userId,
    amount,
    timestamp: new Date(),
  })

  return { success: true, transactionId }
}
```

<details>
<summary>Solution</summary>

```typescript
interface PaymentRequest {
  // Client provides idempotency key
  idempotencyKey: string
  userId: string
  amount: number
}

interface PaymentResult {
  success: boolean
  transactionId?: string
  reason?: string
}

const processPayment = async (
  request: PaymentRequest
): Promise<PaymentResult> => {
  // Check for existing transaction with this idempotency key
  const existing = await database.findTransactionByIdempotencyKey(
    request.idempotencyKey
  )

  if (existing) {
    // Already processed - return cached result
    return {
      success: true,
      transactionId: existing.id,
    }
  }

  // Use database transaction for atomicity
  return await database.transaction(async (tx) => {
    // Lock the user row for update
    const user = await tx.getUserForUpdate(request.userId)

    if (user.balance < request.amount) {
      return { success: false, reason: 'Insufficient funds' }
    }

    // Generate transaction ID deterministically from idempotency key
    // This ensures retries don't create duplicate IDs
    const transactionId = `txn-${request.idempotencyKey}`

    // Atomic updates within transaction
    await tx.updateUser(request.userId, {
      balance: user.balance - request.amount,
    })

    await tx.insertTransaction({
      id: transactionId,
      idempotencyKey: request.idempotencyKey,
      userId: request.userId,
      amount: request.amount,
      timestamp: new Date(),
    })

    return { success: true, transactionId }
  })
}

// Client usage:
const payForOrder = async (order: Order): Promise<PaymentResult> => {
  const request: PaymentRequest = {
    // Idempotency key derived from order - same order = same key
    idempotencyKey: `payment-${order.id}-${order.total}`,
    userId: order.userId,
    amount: order.total,
  }

  // Safe to retry on network failure
  return await retryWithBackoff(() => processPayment(request), {
    maxAttempts: 3,
    backoffMs: 1000,
  })
}
```
</details>

### Exercise 4: Use IO Type

Convert these functions to use the IO type from fp-ts.

```typescript
// Convert these to return IO instead of executing immediately

const logMessage = (msg: string): void => {
  console.log(msg)
}

const getRandomInt = (max: number): number => {
  return Math.floor(Math.random() * max)
}

const getCurrentTimestamp = (): string => {
  return new Date().toISOString()
}

// Then compose them into a program that:
// 1. Gets current timestamp
// 2. Gets a random number 0-100
// 3. Logs: "[timestamp] Random number: X"
```

<details>
<summary>Solution</summary>

```typescript
import { pipe } from 'fp-ts/function'
import * as IO from 'fp-ts/IO'

// Convert to IO - effects are deferred
const logMessage = (msg: string): IO.IO<void> =>
  () => console.log(msg)

const getRandomInt = (max: number): IO.IO<number> =>
  () => Math.floor(Math.random() * max)

const getCurrentTimestamp: IO.IO<string> =
  () => new Date().toISOString()

// Compose into a program
const logRandomNumber: IO.IO<void> = pipe(
  // Get timestamp
  getCurrentTimestamp,
  IO.bindTo('timestamp'),
  // Get random number
  IO.bind('randomNum', () => getRandomInt(100)),
  // Format and log message
  IO.chain(({ timestamp, randomNum }) =>
    logMessage(`[${timestamp}] Random number: ${randomNum}`)
  )
)

// Alternative composition style
const logRandomNumber2: IO.IO<void> = pipe(
  IO.Do,
  IO.bind('timestamp', () => getCurrentTimestamp),
  IO.bind('randomNum', () => getRandomInt(100)),
  IO.chain(({ timestamp, randomNum }) =>
    logMessage(`[${timestamp}] Random number: ${randomNum}`)
  )
)

// Nothing has happened yet! Program is just data.
// Execute when ready:
logRandomNumber()
// Logs: "[2024-01-15T10:30:00.000Z] Random number: 42"

// Can execute multiple times, gets different results each time:
logRandomNumber()
logRandomNumber()
```
</details>

---

## Summary

| Concept | Key Idea | Benefit |
|---------|----------|---------|
| Side Effects | Inputs/outputs beyond parameters and return value | Understanding what makes code impure |
| Pure Core, Impure Shell | Business logic pure, effects at boundaries | Testable, predictable core |
| Idempotence | Same operation multiple times = same result | Safe retries, async safety |
| IO Type | Deferred synchronous effects as values | Composable effect descriptions |
| Quarantine | Isolate impure code in specific modules | Clear separation of concerns |
| Dependency Injection | Accept effects as parameters | Testable, configurable functions |

## Key Takeaways

1. **Side effects are unavoidable** - the goal is control, not elimination
2. **Push effects to the edges** - pure core, thin impure shell
3. **Make dependencies explicit** - inject effects, don't hide them
4. **Design for idempotence** - especially for async operations
5. **Use types to track effects** - IO, Task, TaskEither make effects visible
6. **Accept controlled impurity** - logging, caching, config loading are often acceptable

## Next Steps

With side effect management understood, you're ready for:
1. **fp-ts Task and TaskEither**: Async effect handling
2. **fp-ts Reader**: Dependency injection via environment
3. **Effect systems**: More advanced effect tracking (Effect-TS)
4. **Testing strategies**: Property-based testing for pure functions

Remember: The functional approach to side effects isn't about purity for its own sake - it's about making your code more predictable, testable, and maintainable. Start with the "pure core, impure shell" pattern and gradually adopt more sophisticated effect handling as needed.
