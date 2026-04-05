---
name: react
description: Travis's React patterns — components, hooks, context, Zustand, React Query, URQL, Tailwind, forms. Use when building React or Next.js UIs.
---

# React Patterns

## Components

Named exports with a `Props` interface defined immediately above the component:

```tsx
import { FC } from 'react'

// components/UserCard.tsx
interface UserCardProps {
  userId: string
  name: string
  email: string
  onEdit: (id: string) => void
}

export const UserCard: FC<UserCardProps> = ({ userId, name, email, onEdit }) => {
  return (
    <div className="flex items-center p-4 border border-gray-100 rounded-lg">
      <div>
        <p className="font-semibold">{name}</p>
        <p className="text-sm text-gray-500">{email}</p>
      </div>
      <button onClick={() => onEdit(userId)}>Edit</button>
    </div>
  )
}
```

Rules:
- Named exports — not `export default` for components
- Props interface defined immediately above the component
- Use `PropsWithChildren<Props>` when children are accepted
- Tailwind for all styling — no inline styles, no CSS modules (unless legacy)
- No prop drilling more than 2 levels — use context or state management
- `useCallback` for all function handlers (not inline arrows)
- `useMemo` wherever derived values are computed

## Custom Hooks

In `hooks/` directory, prefixed with `use`:

```ts
// hooks/useStore.ts
export const useStore = () => {
  const context = useContext(StoreContext)
  if (!context) throw new Error('useStore must be used within StoreProvider')
  return context
}
```

## Context

```tsx
// context/store-context.tsx
interface StoreContextData {
  storeProperties: StoreProperties | null
  isLoading: boolean
}

const StoreContext = createContext<StoreContextData | null>(null)

export const StoreProvider = ({ children }: { children: React.ReactNode }) => {
  const [storeProperties, setStoreProperties] = useState<StoreProperties | null>(null)

  return (
    <StoreContext.Provider value={{ storeProperties, isLoading: false }}>
      {children}
    </StoreContext.Provider>
  )
}

export const useStore = () => {
  const ctx = useContext(StoreContext)
  if (!ctx) throw new Error('useStore must be used within StoreProvider')
  return ctx
}
```

## Tailwind CSS

Always Tailwind for styling. Class composition with `classnames` or `clsx`:

```tsx
import classNames from 'classnames'

<div className={classNames(
  'flex items-center py-2 my-3 border-b border-gray-100',
  { 'opacity-50': isDisabled },
  className,
)} />
```

Semantic ordering in className — layout → spacing → typography → color → other:
```tsx
className="flex items-center gap-2 px-4 py-2 text-sm font-medium text-gray-700 bg-white border rounded-lg hover:bg-gray-50"
```

## State Management: Zustand

For persistent client state (auth, UI state) in Next.js/React apps:

```ts
import { create } from 'zustand'
import { persist, devtools } from 'zustand/middleware'

interface AuthStore {
  user: User | null
  isHydrated: boolean
  setUser: (user: User | null) => void
}

export const useAuthStore = create<AuthStore>()(
  devtools(
    persist(
      (set) => ({
        user: null,
        isHydrated: false,
        setUser: (user) => set({ user }),
      }),
      { name: 'auth-store', onRehydrateStorage: () => (state) => state && set({ isHydrated: true }) },
    ),
  ),
)
```

## Server State: TanStack React Query

For data fetching, caching, and mutations. Pairs with `graphql-request` typed client:

```tsx
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query'
import { graphql } from '@/utils/graphql-client'  // graphql-codegen SDK

function useOrders() {
  return useQuery({
    queryKey: ['orders'],
    queryFn: () => graphql.GetOrders(),
    staleTime: 30_000,
  })
}

function useCreateOrder() {
  const queryClient = useQueryClient()
  return useMutation({
    mutationFn: graphql.CreateOrder,
    onSuccess: () => queryClient.invalidateQueries({ queryKey: ['orders'] }),
  })
}
```

## URQL (GraphQL in React)

Used in some React apps as an alternative to React Query + graphql-request:

```tsx
import { useQuery, useMutation, gql } from 'urql'

const GET_USER = gql`
  query GetUser($id: ID!) {
    user(id: $id) { id name email }
  }
`

const [{ data, fetching, error }] = useQuery({ query: GET_USER, variables: { id: userId } })
const [, updateUser] = useMutation(UPDATE_USER)
```

## Forms

Controlled components with typed state + Zod validation before submit:

```tsx
const [formData, setFormData] = useState<{ name: string; email: string }>({ name: '', email: '' })

const handleChange = (field: keyof typeof formData) => (e: React.ChangeEvent<HTMLInputElement>) => {
  setFormData((prev) => ({ ...prev, [field]: e.target.value }))
}

const handleSubmit = () => {
  const result = schema.safeParse(formData)
  if (!result.success) {
    setErrors(result.error.flatten().fieldErrors)
    return
  }
  // submit result.data
}
```

## Loading & Error States

Always handle explicitly — never render data without a loading/error guard:

```tsx
if (loading) return <LoadingScreen />
if (error) return <ErrorBoundary error={error} />
if (!data) return null

return <DataTable data={data} />
```

## UI Libraries

| Project type | Library |
|---|---|
| Modern React apps | Tailwind CSS only (custom components) |
| Older React apps | `@headlessui/react` + `@heroicons/react` |
| Dashboard apps | Chakra UI or Naive UI |

## Inertia.js (AdonisJS + React)

For server-driven pages where AdonisJS controllers render React views:

```ts
// AdonisJS controller
return inertia.render('Dashboard', { user, stats })
```

```tsx
// pages/Dashboard.tsx — Inertia page (default export required)
export default function Dashboard({ user, stats }: { user: User; stats: Stats }) {
  return <div>...</div>
}
```

## TypeScript in React

**Event handlers** — always use React's event types:
```ts
const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => { }
const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => { e.preventDefault() }
const handleClick = (e: React.MouseEvent<HTMLButtonElement>) => { }
const handleKeyDown = (e: React.KeyboardEvent<HTMLInputElement>) => { }
```

**Refs**:
```ts
const inputRef = useRef<HTMLInputElement>(null)
const divRef = useRef<HTMLDivElement>(null)
// Access: inputRef.current?.focus()
```

**Children typing**:
```tsx
// Prefer explicit prop over ReactNode where possible
interface Props {
  children: React.ReactNode       // any renderable content
  header: React.ReactElement      // must be a JSX element
  label: string                   // prefer string over ReactNode for text
}

// Shorthand helper
const Layout: FC<PropsWithChildren<{ title: string }>> = ({ children, title }) => { }
```

**Generic components**:
```tsx
interface ListProps<T> {
  items: T[]
  renderItem: (item: T) => React.ReactNode
  keyExtractor: (item: T) => string
}

function List<T>({ items, renderItem, keyExtractor }: ListProps<T>) {
  return <ul>{items.map(item => <li key={keyExtractor(item)}>{renderItem(item)}</li>)}</ul>
}
```

**`forwardRef`**:
```tsx
interface InputProps extends React.InputHTMLAttributes<HTMLInputElement> {
  label: string
}

export const Input = forwardRef<HTMLInputElement, InputProps>(({ label, ...props }, ref) => (
  <label>
    {label}
    <input ref={ref} {...props} />
  </label>
))
Input.displayName = 'Input'
```

**`useReducer`**:
```ts
type State = { count: number; status: 'idle' | 'loading' | 'error' }
type Action = { type: 'increment' } | { type: 'setLoading' } | { type: 'setError' }

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'increment': return { ...state, count: state.count + 1 }
    case 'setLoading': return { ...state, status: 'loading' }
    case 'setError': return { ...state, status: 'error' }
  }
}

const [state, dispatch] = useReducer(reducer, { count: 0, status: 'idle' })
```

<!--
Source references:
- https://react.dev/reference/react
- https://tanstack.com/query/latest
- https://zustand.docs.pmnd.rs
- https://formidable.com/open-source/urql/
-->
