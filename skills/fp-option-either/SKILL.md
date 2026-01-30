---
name: fp-ts Option and Either
description: Functional error handling and nullable value management using fp-ts Option and Either types
version: 1.0.0
author: kadu
tags:
  - fp-ts
  - functional-programming
  - typescript
  - error-handling
  - option
  - either
  - monads
---

# fp-ts Option and Either Guide

This skill covers practical usage of `Option` and `Either` from fp-ts for safer TypeScript code.

## When to Use Option vs Either

### Use Option when:
- A value may or may not exist (nullable/undefined scenarios)
- You don't need to know WHY a value is missing
- Working with optional fields, array lookups, or dictionary access

### Use Either when:
- An operation can fail and you need error information
- Replacing try-catch blocks
- You need to communicate different failure reasons
- Building validation pipelines

## Imports

```typescript
// Option imports
import * as O from 'fp-ts/Option'
import { pipe } from 'fp-ts/function'

// Either imports
import * as E from 'fp-ts/Either'
import { pipe } from 'fp-ts/function'

// Both together
import * as O from 'fp-ts/Option'
import * as E from 'fp-ts/Either'
import { pipe, flow } from 'fp-ts/function'
```

## Option: Handling Nullable Values

### Converting Nullable Values to Option

```typescript
import * as O from 'fp-ts/Option'
import { pipe } from 'fp-ts/function'

// fromNullable: converts null/undefined to None, otherwise Some
const maybeUser = O.fromNullable(getUserById(id)) // Option<User>

// fromPredicate: creates Some if predicate passes, None otherwise
const positiveNumber = O.fromPredicate((n: number) => n > 0)(value)

// Manual construction
const some = O.some(42)        // Some(42)
const none = O.none            // None
```

### Extracting Values from Option

```typescript
// getOrElse: provide a default value
const username = pipe(
  maybeUser,
  O.map(user => user.name),
  O.getOrElse(() => 'Anonymous')
)

// getOrElseW: wider type for default (when types differ)
const result = pipe(
  maybeNumber,
  O.getOrElseW(() => 'not found' as const)
) // number | 'not found'

// fold/match: handle both cases explicitly
const greeting = pipe(
  maybeUser,
  O.fold(
    () => 'Hello, stranger!',           // None case
    (user) => `Hello, ${user.name}!`    // Some case
  )
)

// Alternative: match (same as fold, more descriptive name)
const greeting = pipe(
  maybeUser,
  O.match(
    () => 'Hello, stranger!',
    (user) => `Hello, ${user.name}!`
  )
)
```

### Transforming Option Values

```typescript
// map: transform the inner value (if present)
const userName = pipe(
  maybeUser,
  O.map(user => user.name)
) // Option<string>

// chain (flatMap): when transformation returns Option
const userEmail = pipe(
  maybeUser,
  O.chain(user => O.fromNullable(user.email))
) // Option<string>

// filter: keep Some only if predicate passes
const adultUser = pipe(
  maybeUser,
  O.filter(user => user.age >= 18)
)
```

### Combining Options

```typescript
// sequenceArray: convert Option<T>[] to Option<T[]>
import { sequenceArray } from 'fp-ts/Option'

const maybeNumbers: O.Option<number>[] = [O.some(1), O.some(2), O.some(3)]
const allNumbers = sequenceArray(maybeNumbers) // Some([1, 2, 3])

const withNone: O.Option<number>[] = [O.some(1), O.none, O.some(3)]
const result = sequenceArray(withNone) // None

// ap: apply Option<(a: A) => B> to Option<A>
import { ap } from 'fp-ts/Option'

const add = (a: number) => (b: number) => a + b
const result = pipe(
  O.some(add),
  ap(O.some(1)),
  ap(O.some(2))
) // Some(3)
```

## Either: Handling Errors

### Converting Try-Catch to Either

```typescript
import * as E from 'fp-ts/Either'
import { pipe } from 'fp-ts/function'

// tryCatch: wraps throwing functions
const parseJSON = (json: string): E.Either<Error, unknown> =>
  E.tryCatch(
    () => JSON.parse(json),
    (error) => error instanceof Error ? error : new Error(String(error))
  )

// Usage
const result = parseJSON('{"valid": "json"}') // Right({ valid: 'json' })
const error = parseJSON('invalid json')        // Left(SyntaxError)

// tryCatchK: creates a function that returns Either
const safeParseJSON = E.tryCatchK(
  JSON.parse,
  (error) => error instanceof Error ? error : new Error(String(error))
)
```

### Creating Either Values

```typescript
// Manual construction
const success = E.right(42)                    // Right(42)
const failure = E.left('Something went wrong') // Left('Something went wrong')

// fromNullable: convert nullable to Either with error
const getUser = (id: string): E.Either<string, User> =>
  pipe(
    findUserById(id),
    E.fromNullable(`User not found: ${id}`)
  )

// fromPredicate: create Right if predicate passes
const validateAge = E.fromPredicate(
  (age: number) => age >= 18,
  (age) => `Age ${age} is below minimum of 18`
)
```

### Extracting Values from Either

```typescript
// fold/match: handle both cases
const message = pipe(
  result,
  E.fold(
    (error) => `Error: ${error.message}`,  // Left case
    (data) => `Success: ${JSON.stringify(data)}` // Right case
  )
)

// getOrElse: provide default for Left case
const value = pipe(
  result,
  E.getOrElse((error) => defaultValue)
)

// getOrElseW: wider type for default
const value = pipe(
  result,
  E.getOrElseW((error) => null)
) // T | null
```

### Transforming Either Values

```typescript
// map: transform Right value
const userAge = pipe(
  getUser(id),
  E.map(user => user.age)
) // Either<string, number>

// mapLeft: transform Left value (error mapping)
const withBetterError = pipe(
  result,
  E.mapLeft(error => new CustomError(error.message))
)

// bimap: transform both sides
const formatted = pipe(
  result,
  E.bimap(
    (error) => `Error: ${error}`,
    (value) => `Value: ${value}`
  )
)

// chain (flatMap): when transformation returns Either
const userProfile = pipe(
  getUser(id),
  E.chain(user => getProfile(user.profileId))
) // Either<string, Profile>

// chainW: chain with wider error type
const result = pipe(
  validateEmail(input),      // Either<ValidationError, string>
  E.chainW(sendEmail)        // Either<NetworkError, Response>
) // Either<ValidationError | NetworkError, Response>
```

### Validation Pattern

```typescript
import * as E from 'fp-ts/Either'
import { pipe } from 'fp-ts/function'

type ValidationError = { field: string; message: string }

const validateEmail = (email: string): E.Either<ValidationError, string> =>
  email.includes('@')
    ? E.right(email)
    : E.left({ field: 'email', message: 'Invalid email format' })

const validatePassword = (password: string): E.Either<ValidationError, string> =>
  password.length >= 8
    ? E.right(password)
    : E.left({ field: 'password', message: 'Password too short' })

// Sequential validation (stops at first error)
const validateUser = (email: string, password: string) =>
  pipe(
    E.Do,
    E.bind('email', () => validateEmail(email)),
    E.bind('password', () => validatePassword(password))
  )

// For collecting all errors, use Validation from fp-ts
import * as A from 'fp-ts/Apply'
import { getSemigroup } from 'fp-ts/NonEmptyArray'

const applicativeValidation = E.getApplicativeValidation(
  getSemigroup<ValidationError>()
)
```

## Common Patterns

### Safe Array Access

```typescript
import * as A from 'fp-ts/Array'
import * as O from 'fp-ts/Option'

// head: safely get first element
const first = A.head([1, 2, 3]) // Some(1)
const empty = A.head([])        // None

// lookup: safely get element by index
const second = A.lookup(1)([1, 2, 3]) // Some(2)
const outOfBounds = A.lookup(10)([1, 2, 3]) // None
```

### Safe Object Property Access

```typescript
import * as R from 'fp-ts/Record'
import * as O from 'fp-ts/Option'

const config: Record<string, string> = { host: 'localhost' }

const host = R.lookup('host')(config)    // Some('localhost')
const missing = R.lookup('port')(config) // None
```

### Converting Between Option and Either

```typescript
import * as O from 'fp-ts/Option'
import * as E from 'fp-ts/Either'

// Option to Either
const toEither = O.toEither(() => 'Value was missing')
const either = pipe(maybeValue, toEither) // Either<string, T>

// Either to Option (discards error)
const toOption = E.toOption
const option = pipe(either, toOption) // Option<T>
```

### Async Operations with TaskEither

```typescript
import * as TE from 'fp-ts/TaskEither'
import { pipe } from 'fp-ts/function'

// Wrap async operations
const fetchUser = (id: string): TE.TaskEither<Error, User> =>
  TE.tryCatch(
    () => fetch(`/api/users/${id}`).then(r => r.json()),
    (error) => error instanceof Error ? error : new Error(String(error))
  )

// Chain async operations
const getUserProfile = (id: string) =>
  pipe(
    fetchUser(id),
    TE.chain(user => fetchProfile(user.profileId)),
    TE.map(profile => profile.displayName)
  )

// Execute
const result = await getUserProfile('123')() // Either<Error, string>
```

## Best Practices

1. **Prefer `pipe` over method chaining** for better composition and tree-shaking

2. **Use `fromNullable` at system boundaries** to convert external nullable values

3. **Use descriptive error types with Either** instead of generic strings

4. **Leverage type inference** - avoid explicit type annotations when TypeScript can infer

5. **Use `chainW` when error types differ** to automatically widen the union

6. **Prefer `fold`/`match` for final extraction** to ensure both cases are handled

## Anti-Patterns to Avoid

### Don't Use isSome/isRight for Control Flow

```typescript
// BAD: loses type narrowing benefits
if (O.isSome(maybeUser)) {
  console.log(maybeUser.value.name)
}

// GOOD: use fold/match
pipe(
  maybeUser,
  O.fold(
    () => console.log('No user'),
    (user) => console.log(user.name)
  )
)
```

### Don't Nest Options/Eithers

```typescript
// BAD: creates Option<Option<T>>
const nested = pipe(
  maybeUser,
  O.map(user => O.fromNullable(user.email))
) // Option<Option<string>>

// GOOD: use chain to flatten
const flat = pipe(
  maybeUser,
  O.chain(user => O.fromNullable(user.email))
) // Option<string>
```

### Don't Use getOrElse Too Early

```typescript
// BAD: extracts value too early, loses composition
const name = pipe(maybeUser, O.getOrElse(() => defaultUser)).name

// GOOD: keep in Option context, extract at the end
const name = pipe(
  maybeUser,
  O.map(user => user.name),
  O.getOrElse(() => 'Unknown')
)
```

### Don't Ignore Left Values

```typescript
// BAD: silently discards error information
const value = pipe(result, E.getOrElse(() => defaultValue))

// GOOD: handle or log errors appropriately
const value = pipe(
  result,
  E.fold(
    (error) => {
      logger.error('Operation failed', error)
      return defaultValue
    },
    (value) => value
  )
)
```

### Don't Mix Paradigms

```typescript
// BAD: mixing try-catch with Either
try {
  const result = pipe(
    parseJSON(input),
    E.chain(validate)
  )
} catch (e) {
  // This defeats the purpose
}

// GOOD: stay in Either world
pipe(
  parseJSON(input),
  E.chain(validate),
  E.fold(
    (error) => handleError(error),
    (value) => handleSuccess(value)
  )
)
```
