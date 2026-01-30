---
name: fp-ts-react
description: Functional programming patterns for React applications using fp-ts, including referential stability, state management, async data handling, and form validation
version: 1.0.0
author: fp-ts-skills
tags:
  - fp-ts
  - react
  - functional-programming
  - typescript
  - hooks
  - state-management
  - async
  - validation
---

# fp-ts with React

This skill covers patterns for integrating fp-ts with React applications, focusing on referential stability, state management, async operations, and form validation.

## Referential Stability with fp-ts-react-stable-hooks

fp-ts data structures like `Option` and `Either` create new object references on every render, breaking React's memoization. Use `fp-ts-react-stable-hooks` to solve this.

### Installation

```bash
npm install fp-ts-react-stable-hooks
# or
yarn add fp-ts-react-stable-hooks
```

### useStableO - Stable Option

```typescript
import { useStableO } from 'fp-ts-react-stable-hooks'
import * as O from 'fp-ts/Option'
import * as Eq from 'fp-ts/Eq'
import * as S from 'fp-ts/string'
import * as N from 'fp-ts/number'

// Basic usage - value is referentially stable
function UserProfile({ userId }: { userId: string }) {
  const [user, setUser] = useStableO<User>(O.none, userEq)

  // user reference only changes when the inner value changes
  // Safe to use in dependency arrays
  useEffect(() => {
    fetchUser(userId).then(u => setUser(O.some(u)))
  }, [userId])

  return pipe(
    user,
    O.fold(
      () => <Loading />,
      (u) => <UserCard user={u} />
    )
  )
}

// Define Eq instance for your types
const userEq: Eq.Eq<User> = Eq.struct({
  id: S.Eq,
  name: S.Eq,
  email: S.Eq
})
```

### useStableE - Stable Either

```typescript
import { useStableE } from 'fp-ts-react-stable-hooks'
import * as E from 'fp-ts/Either'

function ValidatedInput() {
  const [value, setValue] = useStableE<ValidationError, string>(
    E.right(''),
    validationErrorEq,
    S.Eq
  )

  const handleChange = (input: string) => {
    setValue(validateInput(input))
  }

  return pipe(
    value,
    E.fold(
      (error) => <InputWithError error={error.message} />,
      (valid) => <Input value={valid} onChange={handleChange} />
    )
  )
}
```

### Custom Stable Hook Pattern

When you need stability without the library:

```typescript
import { useRef, useState, useCallback } from 'react'
import * as O from 'fp-ts/Option'
import * as Eq from 'fp-ts/Eq'

function useStableOption<A>(
  initial: O.Option<A>,
  eq: Eq.Eq<A>
): [O.Option<A>, (next: O.Option<A>) => void] {
  const [value, setValue] = useState(initial)
  const valueRef = useRef(value)

  const optionEq = O.getEq(eq)

  const stableSetValue = useCallback((next: O.Option<A>) => {
    if (!optionEq.equals(valueRef.current, next)) {
      valueRef.current = next
      setValue(next)
    }
  }, [])

  return [value, stableSetValue]
}
```

## Managing Nullable State with Option

### Basic Option State

```typescript
import * as O from 'fp-ts/Option'
import { pipe } from 'fp-ts/function'

interface User {
  id: string
  name: string
  email: O.Option<string>
}

function UserSettings() {
  const [selectedUser, setSelectedUser] = useState<O.Option<User>>(O.none)

  const handleSelect = (user: User) => setSelectedUser(O.some(user))
  const handleClear = () => setSelectedUser(O.none)

  return (
    <div>
      {pipe(
        selectedUser,
        O.fold(
          () => <UserSelector onSelect={handleSelect} />,
          (user) => (
            <div>
              <UserDetails user={user} />
              <button onClick={handleClear}>Clear</button>
            </div>
          )
        )
      )}
    </div>
  )
}
```

### Option with Default Values

```typescript
import * as O from 'fp-ts/Option'
import { pipe } from 'fp-ts/function'

function ThemeSelector() {
  const [customTheme, setCustomTheme] = useState<O.Option<Theme>>(O.none)

  const activeTheme = pipe(
    customTheme,
    O.getOrElse(() => defaultTheme)
  )

  return <ThemeProvider theme={activeTheme}>...</ThemeProvider>
}
```

### Chaining Optional Values

```typescript
function UserAvatar({ userId }: { userId: O.Option<string> }) {
  const [users] = useUsers()

  const avatarUrl = pipe(
    userId,
    O.chain(id => O.fromNullable(users[id])),
    O.chain(user => user.avatarUrl),
    O.getOrElse(() => '/default-avatar.png')
  )

  return <img src={avatarUrl} />
}
```

## Async Data with TaskEither

### Basic Data Fetching Hook

```typescript
import * as TE from 'fp-ts/TaskEither'
import * as E from 'fp-ts/Either'
import { pipe } from 'fp-ts/function'

interface FetchError {
  type: 'network' | 'parse' | 'validation'
  message: string
}

function useFetch<A>(
  task: TE.TaskEither<FetchError, A>
): {
  data: E.Either<FetchError, A> | null
  loading: boolean
  refetch: () => void
} {
  const [state, setState] = useState<{
    data: E.Either<FetchError, A> | null
    loading: boolean
  }>({ data: null, loading: true })

  const execute = useCallback(async () => {
    setState(s => ({ ...s, loading: true }))
    const result = await task()
    setState({ data: result, loading: false })
  }, [task])

  useEffect(() => {
    execute()
  }, [execute])

  return { ...state, refetch: execute }
}

// Usage
const fetchUser = (id: string): TE.TaskEither<FetchError, User> =>
  pipe(
    TE.tryCatch(
      () => fetch(`/api/users/${id}`).then(r => r.json()),
      (error): FetchError => ({ type: 'network', message: String(error) })
    ),
    TE.chain(data =>
      pipe(
        validateUser(data),
        E.mapLeft((e): FetchError => ({ type: 'validation', message: e })),
        TE.fromEither
      )
    )
  )

function UserProfile({ userId }: { userId: string }) {
  const fetchTask = useMemo(() => fetchUser(userId), [userId])
  const { data, loading, refetch } = useFetch(fetchTask)

  if (loading) return <Spinner />

  return pipe(
    data,
    E.fold(
      () => null,
      E.fold(
        (error) => <ErrorMessage error={error} onRetry={refetch} />,
        (user) => <UserCard user={user} />
      )
    )
  ) ?? <Spinner />
}
```

### Composing Multiple Async Operations

```typescript
import * as TE from 'fp-ts/TaskEither'
import * as A from 'fp-ts/Array'
import { sequenceT } from 'fp-ts/Apply'

const fetchUserWithPosts = (userId: string): TE.TaskEither<FetchError, UserWithPosts> =>
  pipe(
    sequenceT(TE.ApplyPar)(
      fetchUser(userId),
      fetchUserPosts(userId)
    ),
    TE.map(([user, posts]) => ({ ...user, posts }))
  )

// Sequential operations
const fetchUserThenPosts = (userId: string): TE.TaskEither<FetchError, UserWithPosts> =>
  pipe(
    fetchUser(userId),
    TE.chain(user =>
      pipe(
        fetchUserPosts(user.id),
        TE.map(posts => ({ ...user, posts }))
      )
    )
  )
```

## RemoteData Pattern

RemoteData explicitly models all states of async data: NotAsked, Loading, Failure, Success.

### RemoteData Type Definition

```typescript
import * as E from 'fp-ts/Either'
import * as O from 'fp-ts/Option'

// Define RemoteData ADT
type RemoteData<E, A> =
  | { readonly _tag: 'NotAsked' }
  | { readonly _tag: 'Loading' }
  | { readonly _tag: 'Failure'; readonly error: E }
  | { readonly _tag: 'Success'; readonly data: A }

// Constructors
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

// Fold/match function
const fold = <E, A, B>(
  onNotAsked: () => B,
  onLoading: () => B,
  onFailure: (e: E) => B,
  onSuccess: (a: A) => B
) => (rd: RemoteData<E, A>): B => {
  switch (rd._tag) {
    case 'NotAsked': return onNotAsked()
    case 'Loading': return onLoading()
    case 'Failure': return onFailure(rd.error)
    case 'Success': return onSuccess(rd.data)
  }
}

// Map function
const map = <A, B>(f: (a: A) => B) =>
  <E>(rd: RemoteData<E, A>): RemoteData<E, B> =>
    isSuccess(rd) ? success(f(rd.data)) : rd

// Chain function
const chain = <E, A, B>(f: (a: A) => RemoteData<E, B>) =>
  (rd: RemoteData<E, A>): RemoteData<E, B> =>
    isSuccess(rd) ? f(rd.data) : rd

// From Either
const fromEither = <E, A>(either: E.Either<E, A>): RemoteData<E, A> =>
  pipe(either, E.fold(failure, success))

// Get or else
const getOrElse = <A>(defaultValue: () => A) =>
  <E>(rd: RemoteData<E, A>): A =>
    isSuccess(rd) ? rd.data : defaultValue()
```

### RemoteData Hook

```typescript
function useRemoteData<E, A>(
  task: () => TE.TaskEither<E, A>,
  deps: React.DependencyList = []
): [RemoteData<E, A>, () => void] {
  const [state, setState] = useState<RemoteData<E, A>>(notAsked)

  const execute = useCallback(async () => {
    setState(loading)
    const result = await task()()
    setState(fromEither(result))
  }, deps)

  return [state, execute]
}

// Usage
function UserList() {
  const [users, fetchUsers] = useRemoteData(
    () => fetchAllUsers(),
    []
  )

  useEffect(() => {
    fetchUsers()
  }, [fetchUsers])

  return pipe(
    users,
    fold(
      () => <button onClick={fetchUsers}>Load Users</button>,
      () => <Spinner />,
      (error) => <ErrorBanner error={error} onRetry={fetchUsers} />,
      (data) => <UserTable users={data} />
    )
  )
}
```

### RemoteData with Refresh

```typescript
function useRemoteDataWithRefresh<E, A>(
  task: () => TE.TaskEither<E, A>,
  deps: React.DependencyList = []
): {
  data: RemoteData<E, A>
  fetch: () => void
  refresh: () => void
  isRefreshing: boolean
} {
  const [state, setState] = useState<RemoteData<E, A>>(notAsked)
  const [isRefreshing, setIsRefreshing] = useState(false)

  const execute = useCallback(async (isRefresh: boolean) => {
    if (isRefresh) {
      setIsRefreshing(true)
    } else {
      setState(loading)
    }

    const result = await task()()
    setState(fromEither(result))
    setIsRefreshing(false)
  }, deps)

  return {
    data: state,
    fetch: () => execute(false),
    refresh: () => execute(true),
    isRefreshing
  }
}
```

## Form Validation with Either

### Validation Types and Combinators

```typescript
import * as E from 'fp-ts/Either'
import * as A from 'fp-ts/Apply'
import * as NEA from 'fp-ts/NonEmptyArray'
import { pipe } from 'fp-ts/function'

// Validation error type
interface ValidationError {
  field: string
  message: string
}

// Accumulated validation (collects all errors)
type Validation<A> = E.Either<NEA.NonEmptyArray<ValidationError>, A>

// Applicative for accumulating errors
const validationApplicative = E.getApplicativeValidation(NEA.getSemigroup<ValidationError>())

// Lift a validation function
const validate = <A>(
  field: string,
  predicate: (a: A) => boolean,
  message: string
) => (value: A): Validation<A> =>
  predicate(value)
    ? E.right(value)
    : E.left(NEA.of({ field, message }))

// Common validators
const required = (field: string) =>
  validate<string>(field, s => s.trim().length > 0, `${field} is required`)

const minLength = (field: string, min: number) =>
  validate<string>(field, s => s.length >= min, `${field} must be at least ${min} characters`)

const maxLength = (field: string, max: number) =>
  validate<string>(field, s => s.length <= max, `${field} must be at most ${max} characters`)

const email = (field: string) =>
  validate<string>(field, s => /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(s), `${field} must be a valid email`)

const matches = (field: string, pattern: RegExp, message: string) =>
  validate<string>(field, s => pattern.test(s), message)

// Combine validators for a single field
const combineValidators = <A>(...validators: Array<(a: A) => Validation<A>>) =>
  (value: A): Validation<A> =>
    pipe(
      validators.map(v => v(value)),
      A.sequenceT(validationApplicative),
      E.map(() => value)
    )
```

### Form Validation Hook

```typescript
interface FormState<T> {
  values: T
  errors: Record<keyof T, string[]>
  touched: Record<keyof T, boolean>
  isValid: boolean
  isSubmitting: boolean
}

interface UseFormValidation<T> {
  state: FormState<T>
  setValue: <K extends keyof T>(field: K, value: T[K]) => void
  setTouched: <K extends keyof T>(field: K) => void
  validate: () => Validation<T>
  handleSubmit: (onSubmit: (values: T) => Promise<void>) => (e: React.FormEvent) => void
  getFieldProps: <K extends keyof T>(field: K) => {
    value: T[K]
    onChange: (e: React.ChangeEvent<HTMLInputElement>) => void
    onBlur: () => void
    error: string | undefined
  }
}

function useFormValidation<T extends Record<string, unknown>>(
  initialValues: T,
  validators: { [K in keyof T]?: (value: T[K]) => Validation<T[K]> }
): UseFormValidation<T> {
  const [values, setValues] = useState(initialValues)
  const [errors, setErrors] = useState<Record<keyof T, string[]>>(
    {} as Record<keyof T, string[]>
  )
  const [touched, setTouched] = useState<Record<keyof T, boolean>>(
    {} as Record<keyof T, boolean>
  )
  const [isSubmitting, setIsSubmitting] = useState(false)

  const validateField = useCallback(<K extends keyof T>(field: K, value: T[K]): string[] => {
    const validator = validators[field]
    if (!validator) return []

    return pipe(
      validator(value),
      E.fold(
        (errs) => errs.map(e => e.message),
        () => []
      )
    )
  }, [validators])

  const validateAll = useCallback((): Validation<T> => {
    const validations = Object.keys(validators).map(field => {
      const key = field as keyof T
      const validator = validators[key]
      if (!validator) return E.right(values[key])
      return validator(values[key])
    })

    const newErrors = {} as Record<keyof T, string[]>
    Object.keys(validators).forEach((field, index) => {
      const key = field as keyof T
      newErrors[key] = pipe(
        validations[index],
        E.fold(
          (errs) => errs.map(e => e.message),
          () => []
        )
      )
    })
    setErrors(newErrors)

    // Check if all validations pass
    const hasErrors = Object.values(newErrors).some(
      (fieldErrors) => (fieldErrors as string[]).length > 0
    )

    return hasErrors
      ? E.left(NEA.of({ field: 'form', message: 'Validation failed' }))
      : E.right(values)
  }, [validators, values])

  const setValue = useCallback(<K extends keyof T>(field: K, value: T[K]) => {
    setValues(v => ({ ...v, [field]: value }))
    if (touched[field]) {
      setErrors(e => ({ ...e, [field]: validateField(field, value) }))
    }
  }, [touched, validateField])

  const setFieldTouched = useCallback(<K extends keyof T>(field: K) => {
    setTouched(t => ({ ...t, [field]: true }))
    setErrors(e => ({ ...e, [field]: validateField(field, values[field]) }))
  }, [values, validateField])

  const handleSubmit = useCallback(
    (onSubmit: (values: T) => Promise<void>) =>
      async (e: React.FormEvent) => {
        e.preventDefault()
        setIsSubmitting(true)

        const result = validateAll()
        await pipe(
          result,
          E.fold(
            () => Promise.resolve(),
            (validValues) => onSubmit(validValues)
          )
        )

        setIsSubmitting(false)
      },
    [validateAll]
  )

  const getFieldProps = useCallback(<K extends keyof T>(field: K) => ({
    value: values[field],
    onChange: (e: React.ChangeEvent<HTMLInputElement>) =>
      setValue(field, e.target.value as T[K]),
    onBlur: () => setFieldTouched(field),
    error: touched[field] && errors[field]?.length > 0 ? errors[field][0] : undefined
  }), [values, errors, touched, setValue, setFieldTouched])

  const isValid = Object.values(errors).every(
    (fieldErrors) => (fieldErrors as string[]).length === 0
  )

  return {
    state: { values, errors, touched, isValid, isSubmitting },
    setValue,
    setTouched: setFieldTouched,
    validate: validateAll,
    handleSubmit,
    getFieldProps
  }
}
```

### Form Example

```typescript
interface SignupForm {
  username: string
  email: string
  password: string
  confirmPassword: string
}

const signupValidators = {
  username: combineValidators(
    required('username'),
    minLength('username', 3),
    maxLength('username', 20)
  ),
  email: combineValidators(
    required('email'),
    email('email')
  ),
  password: combineValidators(
    required('password'),
    minLength('password', 8),
    matches('password', /[A-Z]/, 'Password must contain uppercase'),
    matches('password', /[0-9]/, 'Password must contain a number')
  )
}

function SignupForm() {
  const form = useFormValidation<SignupForm>(
    { username: '', email: '', password: '', confirmPassword: '' },
    signupValidators
  )

  const onSubmit = async (values: SignupForm) => {
    await api.signup(values)
  }

  return (
    <form onSubmit={form.handleSubmit(onSubmit)}>
      <FormField label="Username" {...form.getFieldProps('username')} />
      <FormField label="Email" {...form.getFieldProps('email')} />
      <FormField label="Password" type="password" {...form.getFieldProps('password')} />

      <button type="submit" disabled={!form.state.isValid || form.state.isSubmitting}>
        {form.state.isSubmitting ? 'Signing up...' : 'Sign Up'}
      </button>
    </form>
  )
}
```

## ReaderTaskEither for Dependency Injection

### Setting Up Dependencies

```typescript
import * as RTE from 'fp-ts/ReaderTaskEither'
import * as TE from 'fp-ts/TaskEither'
import { pipe } from 'fp-ts/function'

// Define dependencies interface
interface AppDependencies {
  apiClient: {
    get: <T>(url: string) => TE.TaskEither<ApiError, T>
    post: <T>(url: string, body: unknown) => TE.TaskEither<ApiError, T>
  }
  storage: {
    get: (key: string) => TE.TaskEither<StorageError, string | null>
    set: (key: string, value: string) => TE.TaskEither<StorageError, void>
  }
  logger: {
    info: (message: string) => void
    error: (message: string, error: unknown) => void
  }
}

type AppError = ApiError | StorageError | ValidationError

// Define operations using ReaderTaskEither
const fetchUser = (id: string): RTE.ReaderTaskEither<AppDependencies, AppError, User> =>
  pipe(
    RTE.ask<AppDependencies>(),
    RTE.chainTaskEitherK(deps => deps.apiClient.get<User>(`/users/${id}`))
  )

const saveUserPreferences = (
  prefs: UserPreferences
): RTE.ReaderTaskEither<AppDependencies, AppError, void> =>
  pipe(
    RTE.ask<AppDependencies>(),
    RTE.chainTaskEitherK(deps =>
      deps.storage.set('preferences', JSON.stringify(prefs))
    )
  )

// Compose operations
const initializeUser = (
  id: string
): RTE.ReaderTaskEither<AppDependencies, AppError, UserWithPrefs> =>
  pipe(
    fetchUser(id),
    RTE.bindTo('user'),
    RTE.bind('preferences', () =>
      pipe(
        RTE.ask<AppDependencies>(),
        RTE.chainTaskEitherK(deps => deps.storage.get('preferences')),
        RTE.map(prefs => prefs ? JSON.parse(prefs) : defaultPreferences)
      )
    ),
    RTE.map(({ user, preferences }) => ({ ...user, preferences }))
  )
```

### React Context for Dependencies

```typescript
import { createContext, useContext, useMemo } from 'react'
import * as RTE from 'fp-ts/ReaderTaskEither'

const DependenciesContext = createContext<AppDependencies | null>(null)

function DependenciesProvider({
  children,
  dependencies
}: {
  children: React.ReactNode
  dependencies: AppDependencies
}) {
  return (
    <DependenciesContext.Provider value={dependencies}>
      {children}
    </DependenciesContext.Provider>
  )
}

function useDependencies(): AppDependencies {
  const deps = useContext(DependenciesContext)
  if (!deps) {
    throw new Error('useDependencies must be used within DependenciesProvider')
  }
  return deps
}

// Hook to run ReaderTaskEither with injected dependencies
function useRTE<E, A>(
  rte: RTE.ReaderTaskEither<AppDependencies, E, A>
): [RemoteData<E, A>, () => void] {
  const deps = useDependencies()

  const [state, setState] = useState<RemoteData<E, A>>(notAsked)

  const execute = useCallback(async () => {
    setState(loading)
    const result = await rte(deps)()
    setState(fromEither(result))
  }, [deps, rte])

  return [state, execute]
}
```

### Usage in Components

```typescript
// App setup
function App() {
  const dependencies: AppDependencies = useMemo(() => ({
    apiClient: createApiClient(config.apiBaseUrl),
    storage: createStorage(),
    logger: createLogger()
  }), [])

  return (
    <DependenciesProvider dependencies={dependencies}>
      <Router>
        <Routes />
      </Router>
    </DependenciesProvider>
  )
}

// Component using RTE
function UserDashboard({ userId }: { userId: string }) {
  const initUser = useMemo(() => initializeUser(userId), [userId])
  const [userData, fetchUser] = useRTE(initUser)

  useEffect(() => {
    fetchUser()
  }, [fetchUser])

  return pipe(
    userData,
    fold(
      () => <WelcomeScreen onStart={fetchUser} />,
      () => <LoadingDashboard />,
      (error) => <ErrorScreen error={error} onRetry={fetchUser} />,
      (user) => <Dashboard user={user} />
    )
  )
}
```

### Testing with Mock Dependencies

```typescript
const mockDependencies: AppDependencies = {
  apiClient: {
    get: jest.fn().mockReturnValue(TE.right({ id: '1', name: 'Test User' })),
    post: jest.fn().mockReturnValue(TE.right(undefined))
  },
  storage: {
    get: jest.fn().mockReturnValue(TE.right(null)),
    set: jest.fn().mockReturnValue(TE.right(undefined))
  },
  logger: {
    info: jest.fn(),
    error: jest.fn()
  }
}

describe('UserDashboard', () => {
  it('fetches and displays user data', async () => {
    render(
      <DependenciesProvider dependencies={mockDependencies}>
        <UserDashboard userId="1" />
      </DependenciesProvider>
    )

    await waitFor(() => {
      expect(screen.getByText('Test User')).toBeInTheDocument()
    })
  })
})
```

## Component Composition with pipe

### Conditional Rendering

```typescript
import { pipe } from 'fp-ts/function'
import * as O from 'fp-ts/Option'

function ConditionalContent({ user }: { user: O.Option<User> }) {
  return pipe(
    user,
    O.fold(
      () => <GuestContent />,
      (u) => pipe(
        u.subscription,
        O.fold(
          () => <FreeUserContent user={u} />,
          (sub) => sub.tier === 'premium'
            ? <PremiumContent user={u} subscription={sub} />
            : <BasicContent user={u} subscription={sub} />
        )
      )
    )
  )
}
```

### List Rendering with fp-ts

```typescript
import * as A from 'fp-ts/Array'
import * as O from 'fp-ts/Option'
import { pipe } from 'fp-ts/function'

function UserList({ users }: { users: User[] }) {
  return (
    <ul>
      {pipe(
        users,
        A.filter(u => u.isActive),
        A.sort(userByNameOrd),
        A.map(user => <UserListItem key={user.id} user={user} />)
      )}
    </ul>
  )
}

// With grouping
function GroupedUserList({ users }: { users: User[] }) {
  const grouped = pipe(
    users,
    NEA.fromArray,
    O.map(NEA.groupBy(u => u.department)),
    O.getOrElse(() => ({}))
  )

  return (
    <div>
      {Object.entries(grouped).map(([dept, deptUsers]) => (
        <section key={dept}>
          <h2>{dept}</h2>
          <UserList users={deptUsers} />
        </section>
      ))}
    </div>
  )
}
```

### Higher-Order Components with pipe

```typescript
import { pipe } from 'fp-ts/function'
import * as O from 'fp-ts/Option'

// HOC that requires authentication
const withAuth = <P extends object>(
  Component: React.ComponentType<P & { user: User }>
): React.FC<P> => {
  return (props: P) => {
    const [currentUser] = useCurrentUser()

    return pipe(
      currentUser,
      O.fold(
        () => <Redirect to="/login" />,
        (user) => <Component {...props} user={user} />
      )
    )
  }
}

// HOC that handles loading state
const withRemoteData = <P extends object, E, A>(
  Component: React.ComponentType<P & { data: A }>,
  useData: () => RemoteData<E, A>,
  ErrorComponent: React.ComponentType<{ error: E; retry: () => void }>
): React.FC<P> => {
  return (props: P) => {
    const [data, fetch] = useData()

    useEffect(() => {
      if (isNotAsked(data)) fetch()
    }, [data, fetch])

    return pipe(
      data,
      fold(
        () => null,
        () => <Loading />,
        (error) => <ErrorComponent error={error} retry={fetch} />,
        (d) => <Component {...props} data={d} />
      )
    )
  }
}
```

## Best Practices

### 1. Always Use Eq Instances for Stability

```typescript
// Define Eq instances for your domain types
const productEq: Eq.Eq<Product> = Eq.struct({
  id: S.Eq,
  name: S.Eq,
  price: N.Eq
})

// Use with stable hooks
const [product, setProduct] = useStableO<Product>(O.none, productEq)
```

### 2. Prefer RemoteData Over Boolean Flags

```typescript
// Avoid
const [data, setData] = useState<User | null>(null)
const [loading, setLoading] = useState(false)
const [error, setError] = useState<Error | null>(null)

// Prefer
const [userData, setUserData] = useState<RemoteData<Error, User>>(notAsked)
```

### 3. Keep TaskEither Operations Pure

```typescript
// Define operations as pure functions
const fetchAndValidateUser = (id: string): TE.TaskEither<AppError, ValidatedUser> =>
  pipe(
    fetchUser(id),
    TE.chain(validateUser),
    TE.chain(enrichWithMetadata)
  )

// Execute in useEffect or event handlers
useEffect(() => {
  fetchAndValidateUser(userId)().then(result => {
    setState(fromEither(result))
  })
}, [userId])
```

### 4. Use ReaderTaskEither for Testable Code

```typescript
// Operations are easily testable with mock dependencies
const operation: RTE.ReaderTaskEither<Deps, Error, Result> = ...

// In tests
const result = await operation(mockDeps)()
expect(E.isRight(result)).toBe(true)
```

### 5. Validate at Boundaries

```typescript
// Validate data at API boundaries
const fetchUser = (id: string): TE.TaskEither<FetchError, User> =>
  pipe(
    TE.tryCatch(() => fetch(`/api/users/${id}`).then(r => r.json()), toNetworkError),
    TE.chain(flow(validateUserSchema, TE.fromEither)) // Validate immediately
  )
```
