---
name: fp-ts Do Notation
description: Master Do notation in fp-ts to write readable, sequential functional code without callback hell. Covers bind, apS, let, bindTo and real-world patterns.
version: 1.0.0
author: fp-ts-skills
tags:
  - fp-ts
  - functional-programming
  - typescript
  - do-notation
  - monads
  - taskEither
  - readerTaskEither
  - composition
---

# fp-ts Do Notation Guide

Do notation is fp-ts's answer to callback hell. It provides a way to write sequential, imperative-looking code while maintaining functional purity and type safety.

## The Problem: Callback Hell in Functional Code

Without Do notation, chaining dependent operations leads to deeply nested code:

```typescript
import { pipe } from 'fp-ts/function'
import * as TE from 'fp-ts/TaskEither'

// BAD: Nested chain hell
const processOrder = (orderId: string) =>
  pipe(
    fetchOrder(orderId),
    TE.chain((order) =>
      pipe(
        fetchUser(order.userId),
        TE.chain((user) =>
          pipe(
            fetchInventory(order.productId),
            TE.chain((inventory) =>
              pipe(
                validateStock(inventory, order.quantity),
                TE.chain((validated) =>
                  pipe(
                    calculatePrice(order, user.discount),
                    TE.chain((price) =>
                      createInvoice(order, user, price) // Lost context of inventory!
                    )
                  )
                )
              )
            )
          )
        )
      )
    )
  )
```

Problems with nested chains:
1. **Poor readability** - Logic is buried in nesting
2. **Lost context** - Earlier values may not be accessible in inner scopes
3. **Difficult refactoring** - Adding/removing steps requires restructuring
4. **Hard to parallelize** - Everything looks sequential

## The Solution: Do Notation

Do notation flattens the structure and keeps all values in scope:

```typescript
import { pipe } from 'fp-ts/function'
import * as TE from 'fp-ts/TaskEither'

// GOOD: Flat, readable Do notation
const processOrder = (orderId: string) =>
  pipe(
    TE.Do,
    TE.bind('order', () => fetchOrder(orderId)),
    TE.bind('user', ({ order }) => fetchUser(order.userId)),
    TE.bind('inventory', ({ order }) => fetchInventory(order.productId)),
    TE.bind('validated', ({ inventory, order }) => validateStock(inventory, order.quantity)),
    TE.bind('price', ({ order, user }) => calculatePrice(order, user.discount)),
    TE.bind('invoice', ({ order, user, price }) => createInvoice(order, user, price))
  )
```

## Core Do Notation Functions

### `Do` - Starting Point

`Do` creates an empty context object `{}` wrapped in your monad:

```typescript
import * as TE from 'fp-ts/TaskEither'
import * as E from 'fp-ts/Either'
import * as O from 'fp-ts/Option'

TE.Do  // TaskEither<never, {}>
E.Do   // Either<never, {}>
O.Do   // Option<{}>
```

### `bindTo` - Initialize with First Value

Use `bindTo` when you already have a value and want to start Do notation:

```typescript
import { pipe } from 'fp-ts/function'
import * as TE from 'fp-ts/TaskEither'

// Instead of:
pipe(
  TE.Do,
  TE.bind('user', () => fetchUser(userId))
)

// Use bindTo for cleaner initialization:
pipe(
  fetchUser(userId),
  TE.bindTo('user'),
  TE.bind('orders', ({ user }) => fetchOrders(user.id))
)
```

`bindTo` is semantically equivalent to `TE.map(user => ({ user }))` but more readable.

### `bind` - Sequential Dependent Operations

Use `bind` when the next operation **depends on previous values**:

```typescript
import { pipe } from 'fp-ts/function'
import * as TE from 'fp-ts/TaskEither'

const getUserWithPosts = (userId: string) =>
  pipe(
    TE.Do,
    TE.bind('user', () => fetchUser(userId)),           // First: get user
    TE.bind('posts', ({ user }) => fetchPosts(user.id)), // Then: use user.id
    TE.bind('comments', ({ posts }) =>                   // Then: use posts
      TE.traverseArray(fetchComments)(posts.map(p => p.id))
    )
  )
```

**Key characteristics of `bind`:**
- Operations execute **sequentially**
- Each step has access to **all previous values**
- Short-circuits on first error (for Either/TaskEither)
- The callback receives the accumulated context object

### `apS` - Parallel Independent Operations

Use `apS` when operations are **independent** and can run in parallel:

```typescript
import { pipe } from 'fp-ts/function'
import * as TE from 'fp-ts/TaskEither'

const getDashboardData = (userId: string) =>
  pipe(
    TE.Do,
    TE.bind('user', () => fetchUser(userId)),
    // These three are INDEPENDENT - use apS for parallel execution
    TE.apS('notifications', fetchNotifications(userId)),
    TE.apS('settings', fetchSettings(userId)),
    TE.apS('recentActivity', fetchRecentActivity(userId))
  )
```

**Key characteristics of `apS`:**
- Operations can execute **in parallel** (with TaskEither)
- The value is computed **immediately** (not lazily)
- No access to previous context values
- Errors are collected or short-circuit depending on the applicative

### `let` - Computed/Derived Values

Use `let` for **synchronous computations** derived from existing values:

```typescript
import { pipe } from 'fp-ts/function'
import * as TE from 'fp-ts/TaskEither'

const processPayment = (orderId: string) =>
  pipe(
    TE.Do,
    TE.bind('order', () => fetchOrder(orderId)),
    TE.bind('user', ({ order }) => fetchUser(order.userId)),
    // Computed values - no async operation needed
    TE.let('subtotal', ({ order }) => order.items.reduce((sum, i) => sum + i.price, 0)),
    TE.let('discount', ({ user, subtotal }) => subtotal * (user.discountPercent / 100)),
    TE.let('total', ({ subtotal, discount }) => subtotal - discount),
    TE.bind('payment', ({ total, user }) => chargeCard(user.paymentMethod, total))
  )
```

**Key characteristics of `let`:**
- Synchronous pure computation
- Has access to all previous values
- Cannot fail (for error types)
- Use for transformations, calculations, formatting

## bind vs apS: When to Use Which

### Decision Guide

| Situation | Use | Reason |
|-----------|-----|--------|
| Next operation needs previous result | `bind` | Sequential dependency |
| Operations are independent | `apS` | Can parallelize |
| Need to transform/compute | `let` | Synchronous, always succeeds |
| Starting with existing value | `bindTo` | Cleaner than Do + bind |

### Performance: Sequential vs Parallel

```typescript
import { pipe } from 'fp-ts/function'
import * as TE from 'fp-ts/TaskEither'

// SLOW: Sequential execution with bind (3 seconds total)
const slowDashboard = pipe(
  TE.Do,
  TE.bind('users', () => fetchUsers()),      // 1 second
  TE.bind('products', () => fetchProducts()), // 1 second (waits for users)
  TE.bind('orders', () => fetchOrders())      // 1 second (waits for products)
)

// FAST: Parallel execution with apS (1 second total)
const fastDashboard = pipe(
  TE.Do,
  TE.apS('users', fetchUsers()),      // 1 second
  TE.apS('products', fetchProducts()), // 1 second (runs in parallel)
  TE.apS('orders', fetchOrders())      // 1 second (runs in parallel)
)
```

### Mixed Pattern: Sequential Then Parallel

```typescript
import { pipe } from 'fp-ts/function'
import * as TE from 'fp-ts/TaskEither'

const getOrderDetails = (orderId: string) =>
  pipe(
    TE.Do,
    // Sequential: need order first
    TE.bind('order', () => fetchOrder(orderId)),
    // Parallel: these only need order.userId and order.productId
    TE.apS('user', pipe(
      TE.Do,
      TE.bind('order', () => fetchOrder(orderId)), // Need to refetch or...
    )),
    // Better pattern: bind first, then parallel for truly independent operations
  )

// BETTER: Restructure for clarity
const getOrderDetailsBetter = (orderId: string) =>
  pipe(
    fetchOrder(orderId),
    TE.bindTo('order'),
    TE.bind('user', ({ order }) => fetchUser(order.userId)),
    // Now these are independent of each other (but dependent on user/order)
    TE.apS('shippingOptions', fetchShippingOptions(orderId)),
    TE.apS('paymentMethods', fetchPaymentMethods(orderId)),
    TE.let('canCheckout', ({ user, shippingOptions }) =>
      user.verified && shippingOptions.length > 0
    )
  )
```

## Real-World Examples

### Example 1: User Registration Flow

```typescript
import { pipe } from 'fp-ts/function'
import * as TE from 'fp-ts/TaskEither'
import * as E from 'fp-ts/Either'

interface RegistrationInput {
  email: string
  password: string
  name: string
}

interface User {
  id: string
  email: string
  name: string
}

const registerUser = (input: RegistrationInput): TE.TaskEither<Error, User> =>
  pipe(
    TE.Do,
    // Validate input (synchronous)
    TE.bind('validated', () => pipe(
      validateEmail(input.email),
      E.chain(() => validatePassword(input.password)),
      E.map(() => input),
      TE.fromEither
    )),
    // Check if email exists (async)
    TE.bind('emailAvailable', ({ validated }) =>
      checkEmailAvailable(validated.email)
    ),
    // Hash password (async, CPU-intensive)
    TE.bind('hashedPassword', ({ validated }) =>
      hashPassword(validated.password)
    ),
    // Create user in database
    TE.bind('user', ({ validated, hashedPassword }) =>
      createUser({
        email: validated.email,
        name: validated.name,
        passwordHash: hashedPassword
      })
    ),
    // Send welcome email (fire and forget, but still in chain)
    TE.chainFirst(({ user }) => sendWelcomeEmail(user.email)),
    // Return just the user
    TE.map(({ user }) => user)
  )
```

### Example 2: E-commerce Checkout

```typescript
import { pipe } from 'fp-ts/function'
import * as TE from 'fp-ts/TaskEither'

interface CheckoutResult {
  orderId: string
  paymentId: string
  estimatedDelivery: Date
}

const checkout = (
  userId: string,
  cartId: string,
  shippingAddressId: string
): TE.TaskEither<CheckoutError, CheckoutResult> =>
  pipe(
    TE.Do,
    // Fetch required data in parallel where possible
    TE.bind('cart', () => fetchCart(cartId)),
    TE.bind('user', () => fetchUser(userId)),
    TE.apS('shippingAddress', fetchAddress(shippingAddressId)),

    // Validate cart has items
    TE.bind('validatedCart', ({ cart }) =>
      cart.items.length === 0
        ? TE.left(new CheckoutError('Cart is empty'))
        : TE.right(cart)
    ),

    // Check inventory for all items (parallel)
    TE.bind('inventoryCheck', ({ validatedCart }) =>
      pipe(
        validatedCart.items,
        TE.traverseArray((item) => checkInventory(item.productId, item.quantity))
      )
    ),

    // Calculate totals (synchronous)
    TE.let('subtotal', ({ validatedCart }) =>
      validatedCart.items.reduce((sum, item) => sum + item.price * item.quantity, 0)
    ),
    TE.let('tax', ({ subtotal, shippingAddress }) =>
      calculateTax(subtotal, shippingAddress.state)
    ),
    TE.let('shippingCost', ({ shippingAddress, validatedCart }) =>
      calculateShipping(shippingAddress, validatedCart.totalWeight)
    ),
    TE.let('total', ({ subtotal, tax, shippingCost }) =>
      subtotal + tax + shippingCost
    ),

    // Process payment
    TE.bind('payment', ({ user, total }) =>
      processPayment(user.defaultPaymentMethod, total)
    ),

    // Create order
    TE.bind('order', ({ user, validatedCart, shippingAddress, payment, total }) =>
      createOrder({
        userId: user.id,
        items: validatedCart.items,
        shippingAddressId: shippingAddress.id,
        paymentId: payment.id,
        total
      })
    ),

    // Reserve inventory
    TE.chainFirst(({ order, validatedCart }) =>
      pipe(
        validatedCart.items,
        TE.traverseArray((item) => reserveInventory(item.productId, item.quantity, order.id))
      )
    ),

    // Clear cart
    TE.chainFirst(({ cart }) => clearCart(cart.id)),

    // Calculate delivery estimate
    TE.let('estimatedDelivery', ({ shippingAddress }) =>
      calculateDeliveryDate(shippingAddress)
    ),

    // Return result
    TE.map(({ order, payment, estimatedDelivery }) => ({
      orderId: order.id,
      paymentId: payment.id,
      estimatedDelivery
    }))
  )
```

### Example 3: ReaderTaskEither with Dependency Injection

```typescript
import { pipe } from 'fp-ts/function'
import * as RTE from 'fp-ts/ReaderTaskEither'
import * as TE from 'fp-ts/TaskEither'

// Define dependencies
interface Deps {
  userRepo: UserRepository
  orderRepo: OrderRepository
  paymentService: PaymentService
  emailService: EmailService
  logger: Logger
}

// Use RTE.Do for dependency-injected workflows
const processRefund = (
  orderId: string,
  reason: string
): RTE.ReaderTaskEither<Deps, RefundError, RefundResult> =>
  pipe(
    RTE.Do,
    // Access dependencies via RTE.asks
    RTE.bind('deps', () => RTE.ask<Deps>()),

    // Fetch order
    RTE.bind('order', ({ deps }) =>
      RTE.fromTaskEither(deps.orderRepo.findById(orderId))
    ),

    // Validate refund is possible
    RTE.bind('validatedOrder', ({ order }) =>
      order.status !== 'completed'
        ? RTE.left(new RefundError('Order not eligible for refund'))
        : RTE.right(order)
    ),

    // Process refund with payment service
    RTE.bind('refund', ({ deps, validatedOrder }) =>
      RTE.fromTaskEither(
        deps.paymentService.refund(validatedOrder.paymentId, validatedOrder.total)
      )
    ),

    // Update order status
    RTE.bind('updatedOrder', ({ deps, validatedOrder, refund }) =>
      RTE.fromTaskEither(
        deps.orderRepo.update(validatedOrder.id, {
          status: 'refunded',
          refundId: refund.id,
          refundReason: reason
        })
      )
    ),

    // Fetch user for email
    RTE.bind('user', ({ deps, validatedOrder }) =>
      RTE.fromTaskEither(deps.userRepo.findById(validatedOrder.userId))
    ),

    // Send notification (fire and forget)
    RTE.chainFirst(({ deps, user, refund }) =>
      RTE.fromTaskEither(
        deps.emailService.sendRefundConfirmation(user.email, refund)
      )
    ),

    // Log the refund
    RTE.chainFirst(({ deps, order, refund }) =>
      RTE.fromTaskEither(
        deps.logger.info('Refund processed', { orderId: order.id, refundId: refund.id })
      )
    ),

    // Return result
    RTE.map(({ refund, updatedOrder }) => ({
      refundId: refund.id,
      orderId: updatedOrder.id,
      amount: refund.amount,
      status: 'completed'
    }))
  )

// Execute with dependencies
const runRefund = (deps: Deps, orderId: string, reason: string) =>
  processRefund(orderId, reason)(deps)()
```

### Example 4: Validation with Accumulated Errors

```typescript
import { pipe } from 'fp-ts/function'
import * as E from 'fp-ts/Either'
import * as TE from 'fp-ts/TaskEither'
import * as A from 'fp-ts/Apply'
import { sequenceS } from 'fp-ts/Apply'

// For parallel validation that accumulates ALL errors (not short-circuit)
// Use Apply.sequenceS instead of Do notation

interface ValidationError {
  field: string
  message: string
}

type ValidationResult<A> = E.Either<ValidationError[], A>

const validateUserInput = (input: unknown): ValidationResult<ValidUser> => {
  const validateField = <A>(
    field: string,
    value: unknown,
    validator: (v: unknown) => E.Either<string, A>
  ): ValidationResult<A> =>
    pipe(
      validator(value),
      E.mapLeft((message) => [{ field, message }])
    )

  // Use sequenceS with validation applicative to collect ALL errors
  return pipe(
    sequenceS(E.getApplicativeValidation(A.getSemigroup<ValidationError>()))({
      email: validateField('email', input.email, validateEmail),
      password: validateField('password', input.password, validatePassword),
      age: validateField('age', input.age, validateAge),
      name: validateField('name', input.name, validateName)
    })
  )
}

// For TaskEither with parallel execution AND error accumulation:
const validateUserAsync = (input: UserInput): TE.TaskEither<ValidationError[], ValidUser> =>
  pipe(
    sequenceS(TE.ApplicativePar)({
      emailUnique: checkEmailUnique(input.email),
      usernameAvailable: checkUsernameAvailable(input.username),
      phoneValid: validatePhoneNumber(input.phone)
    }),
    TE.map(({ emailUnique, usernameAvailable, phoneValid }) => ({
      ...input,
      emailVerified: emailUnique,
      usernameVerified: usernameAvailable,
      phoneVerified: phoneValid
    }))
  )
```

### Example 5: Complex Data Aggregation

```typescript
import { pipe } from 'fp-ts/function'
import * as TE from 'fp-ts/TaskEither'
import * as A from 'fp-ts/Array'

interface DashboardData {
  user: User
  stats: UserStats
  recentOrders: Order[]
  recommendations: Product[]
  notifications: Notification[]
}

const loadDashboard = (userId: string): TE.TaskEither<Error, DashboardData> =>
  pipe(
    TE.Do,
    // First, get user (required for everything)
    TE.bind('user', () => fetchUser(userId)),

    // These are all independent - parallel execution
    TE.apS('stats', fetchUserStats(userId)),
    TE.apS('recentOrders', fetchRecentOrders(userId)),
    TE.apS('notifications', fetchNotifications(userId)),

    // Recommendations depend on user preferences
    TE.bind('recommendations', ({ user }) =>
      fetchRecommendations(user.preferences)
    ),

    // Enhance orders with product details (depends on recentOrders)
    TE.bind('ordersWithProducts', ({ recentOrders }) =>
      pipe(
        recentOrders,
        A.map((order) =>
          pipe(
            fetchProductDetails(order.productId),
            TE.map((product) => ({ ...order, product }))
          )
        ),
        TE.sequenceArray
      )
    ),

    // Compute derived data
    TE.let('unreadCount', ({ notifications }) =>
      notifications.filter((n) => !n.read).length
    ),
    TE.let('totalSpent', ({ recentOrders }) =>
      recentOrders.reduce((sum, o) => sum + o.total, 0)
    ),

    // Return final shape
    TE.map(({ user, stats, ordersWithProducts, recommendations, notifications }) => ({
      user,
      stats,
      recentOrders: ordersWithProducts,
      recommendations,
      notifications
    }))
  )
```

## Common Patterns and Tips

### Pattern 1: Early Return / Guard Clauses

```typescript
import { pipe } from 'fp-ts/function'
import * as TE from 'fp-ts/TaskEither'

const deleteAccount = (userId: string, confirmationCode: string) =>
  pipe(
    TE.Do,
    TE.bind('user', () => fetchUser(userId)),
    // Guard: Check confirmation code
    TE.bind('confirmed', ({ user }) =>
      confirmationCode === user.deleteConfirmationCode
        ? TE.right(true)
        : TE.left(new Error('Invalid confirmation code'))
    ),
    // Guard: Check no pending orders
    TE.bind('pendingOrders', ({ user }) => fetchPendingOrders(user.id)),
    TE.bind('canDelete', ({ pendingOrders }) =>
      pendingOrders.length === 0
        ? TE.right(true)
        : TE.left(new Error('Cannot delete account with pending orders'))
    ),
    // Proceed with deletion
    TE.bind('deleted', ({ user }) => deleteUserAccount(user.id))
  )
```

### Pattern 2: Optional Operations with chainFirst

Use `chainFirst` when you want to perform a side effect but keep the original value:

```typescript
import { pipe } from 'fp-ts/function'
import * as TE from 'fp-ts/TaskEither'

const createPost = (input: PostInput) =>
  pipe(
    TE.Do,
    TE.bind('post', () => savePost(input)),
    // Log creation (side effect, ignore result)
    TE.chainFirst(({ post }) => logPostCreation(post.id)),
    // Notify followers (side effect, ignore result)
    TE.chainFirst(({ post }) => notifyFollowers(post.authorId, post.id)),
    // Index for search (side effect, ignore result)
    TE.chainFirst(({ post }) => indexForSearch(post)),
    // Return just the post
    TE.map(({ post }) => post)
  )
```

### Pattern 3: Conditional Binding

```typescript
import { pipe } from 'fp-ts/function'
import * as TE from 'fp-ts/TaskEither'
import * as O from 'fp-ts/Option'

const processOrder = (orderId: string, promoCode?: string) =>
  pipe(
    TE.Do,
    TE.bind('order', () => fetchOrder(orderId)),
    // Conditionally apply promo code
    TE.bind('discount', ({ order }) =>
      promoCode
        ? validatePromoCode(promoCode, order.total)
        : TE.right(0)
    ),
    TE.let('finalTotal', ({ order, discount }) => order.total - discount)
  )
```

### Pattern 4: Working with Arrays in Do

```typescript
import { pipe } from 'fp-ts/function'
import * as TE from 'fp-ts/TaskEither'
import * as A from 'fp-ts/Array'

const processOrders = (orderIds: string[]) =>
  pipe(
    TE.Do,
    // Fetch all orders in parallel
    TE.bind('orders', () =>
      pipe(
        orderIds,
        A.map(fetchOrder),
        TE.sequenceArray
      )
    ),
    // Process each order
    TE.bind('processed', ({ orders }) =>
      pipe(
        orders,
        A.map(processOrder),
        TE.sequenceArray
      )
    ),
    // Aggregate results
    TE.let('summary', ({ processed }) => ({
      total: processed.length,
      successful: processed.filter((p) => p.status === 'success').length,
      failed: processed.filter((p) => p.status === 'failed').length
    }))
  )
```

## Performance Considerations

### 1. Prefer apS for Independent Operations

```typescript
// SLOW: 3 sequential API calls
pipe(
  TE.Do,
  TE.bind('a', () => fetchA()), // 100ms
  TE.bind('b', () => fetchB()), // 100ms
  TE.bind('c', () => fetchC())  // 100ms
) // Total: 300ms

// FAST: 3 parallel API calls
pipe(
  TE.Do,
  TE.apS('a', fetchA()), // 100ms
  TE.apS('b', fetchB()), // 100ms (parallel)
  TE.apS('c', fetchC())  // 100ms (parallel)
) // Total: ~100ms
```

### 2. Use let for Pure Computations

```typescript
// WRONG: Using bind for pure computation
TE.bind('total', ({ items }) => TE.right(items.reduce((s, i) => s + i.price, 0)))

// RIGHT: Using let for pure computation
TE.let('total', ({ items }) => items.reduce((s, i) => s + i.price, 0))
```

### 3. Batch Database Operations

```typescript
// SLOW: N+1 queries
pipe(
  TE.Do,
  TE.bind('orders', () => fetchOrders(userId)),
  TE.bind('products', ({ orders }) =>
    pipe(
      orders,
      A.map((o) => fetchProduct(o.productId)), // N queries!
      TE.sequenceArray
    )
  )
)

// FAST: Batch query
pipe(
  TE.Do,
  TE.bind('orders', () => fetchOrders(userId)),
  TE.bind('products', ({ orders }) =>
    fetchProductsByIds(orders.map((o) => o.productId)) // 1 query
  )
)
```

### 4. Avoid Rebuilding Context

```typescript
// INEFFICIENT: Rebuilding large context
pipe(
  TE.Do,
  TE.bind('hugeData', () => fetchHugeData()),
  TE.map(({ hugeData }) => ({ hugeData, processed: true })), // Copies hugeData
  TE.bind('more', () => fetchMore()) // hugeData still in context
)

// BETTER: Extract what you need early
pipe(
  TE.Do,
  TE.bind('hugeData', () => fetchHugeData()),
  TE.let('summary', ({ hugeData }) => summarize(hugeData)), // Extract summary
  TE.map(({ summary }) => summary) // Drop hugeData from context
)
```

## Summary

Do notation transforms deeply nested callback chains into flat, readable pipelines:

| Function | Purpose | When to Use |
|----------|---------|-------------|
| `Do` | Start empty context | Beginning of chain |
| `bindTo` | Start with value | When you have initial value |
| `bind` | Sequential operation | Depends on previous values |
| `apS` | Parallel operation | Independent of other values |
| `let` | Pure computation | Derive values synchronously |
| `chainFirst` | Side effect | Fire-and-forget operations |

**Key principles:**
1. Use `bind` for dependencies, `apS` for independence
2. Use `let` for pure computations, never `bind` with `TE.right`
3. Keep the context lean - don't accumulate unnecessary data
4. Combine with `sequenceArray`/`traverseArray` for collections
5. Use `chainFirst` for side effects that shouldn't affect the result

Do notation is the key to writing maintainable fp-ts code. Master it, and functional programming becomes as readable as imperative code while retaining all its benefits.
