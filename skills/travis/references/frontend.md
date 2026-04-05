---
name: frontend
description: Travis's frontend patterns — React component style, Vue/Nuxt conventions, Tailwind CSS, state management. Use when building UI components, pages, or frontend features.
---

# Frontend Patterns

## React Components

Components use named exports with a `Props` interface type defined above, making use of React types for functional components and any helper types like `PropsWithChildren<>` where needed:

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
- Named exports, not `export default` for components
- Props type defined immediately above the component
- Always use provided React types/helpers when possible
- Tailwind for all styling — no inline styles, no CSS modules (unless legacy)
- No prop drilling more than 2 levels — use context or state management
- Always use `useCallback` for function handlers, instead of inline functions
- Always use `useMemo` where possible

## React Hooks

Custom hooks in `hooks/` directory, prefixed with `use`:

```tsx
// hooks/useStore.ts
export const useStore = () => {
  const context = useContext(StoreContext)
  if (!context) throw new Error('useStore must be used within StoreProvider')
  return context
}
```

## React Context

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

Prefer semantic grouping in className — layout first, then visual:
```tsx
// Layout → Spacing → Typography → Color → Other
className="flex items-center gap-2 px-4 py-2 text-sm font-medium text-gray-700 bg-white border rounded-lg hover:bg-gray-50"
```

## URQL Client / GraphQL in React

```tsx
import { useQuery, useMutation, gql } from 'urql'

const GET_USER = gql`
  query GetUser($id: ID!) {
    user(id: $id) { id name email }
  }
`

const { data, loading, error } = useQuery(GET_USER, {
  variables: { id: userId },
})

const [updateUser] = useMutation(UPDATE_USER, {
  refetchQueries: [GET_USER],
})
```

## Vue 3 / Nuxt 3

Use `<script setup>` syntax — never Options API in new code:

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

interface Props {
  userId: string
  name: string
}

const props = defineProps<Props>()
const emit = defineEmits<{ update: [id: string] }>()

const isEditing = ref(false)
const displayName = computed(() => props.name.trim())
</script>

<template>
  <div class="flex items-center gap-2">
    <span>{{ displayName }}</span>
    <button @click="emit('update', props.userId)">Edit</button>
  </div>
</template>
```

**Nuxt composables pattern:**
```ts
// composables/useUser.ts
export function useUser (userId: string) {
  const { data, pending, error } = await useFetch(`/api/users/${userId}`)
  return { user: data, isLoading: pending, error }
}
```

**Nuxt server API routes:**
```ts
// server/api/users/[id].get.ts
export default defineEventHandler(async (event) => {
  const { id } = getRouterParams(event)
  const user = await db.users.find(id)
  if (!user) throw createError({ statusCode: 404, message: 'User not found' })
  return user
})
```

Use `@nuxt/ui` component library with Nuxt 3.

## UI Libraries

| Project type | Library |
|---|---|
| Nuxt 3 apps | `@nuxt/ui` |
| Older React apps | `@headlessui/react` + `@heroicons/react` |
| Dashboard apps | Naive UI or Headless UI |

Always pair with Tailwind CSS.

## Forms

Use controlled components with typed state:

```tsx
const [formData, setFormData] = useState<{ name: string; email: string }>({
  name: '',
  email: '',
})

const handleChange = (field: keyof typeof formData) => (e: React.ChangeEvent<HTMLInputElement>) => {
  setFormData((prev) => ({ ...prev, [field]: e.target.value }))
}
```

Use Zod for validation before submitting:
```ts
const result = schema.safeParse(formData)
if (!result.success) {
  setErrors(result.error.flatten().fieldErrors)
  return
}
```

## Loading & Error States

Always handle loading and error states explicitly:
```tsx
if (loading) return <LoadingScreen />
if (error) return <ErrorBoundary error={error} />
if (!data) return null

return <DataTable data={data} />
```

## Inertia.js (AdonisJS + React/Vue)

For server-driven React/Vue pages with AdonisJS:
```ts
// Controller
return inertia.render('Dashboard', { user, stats })
```

```tsx
// pages/Dashboard.tsx
export default function Dashboard({ user, stats }: { user: User; stats: Stats }) {
  return <div>...</div>
}
```
