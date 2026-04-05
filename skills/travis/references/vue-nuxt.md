---
name: vue-nuxt
description: Travis's Vue 3 and Nuxt 3 patterns — script setup, composables, server routes, @nuxt/ui, NuxtHub, nuxt-auth-utils, Storyblok. Use when building Vue or Nuxt apps.
---

# Vue 3 / Nuxt 3 Patterns

## Vue Components

Always `<script setup>` — never Options API in new code:

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

Rules:
- `<script setup lang="ts">` always
- Props via `defineProps<Props>()` with an inline or above-defined interface
- Emits via `defineEmits<{ eventName: [argType] }>()`
- Use `@nuxt/ui` components for UI — no custom base components when a `U*` component fits

## Composables

In `composables/` directory, prefixed with `use`. Auto-imported by Nuxt:

```ts
// composables/useUser.ts
export function useUser(userId: string) {
  const { data, pending, error } = useFetch(`/api/users/${userId}`)
  return { user: data, isLoading: pending, error }
}
```

For async data with SSR support:
```ts
const { data: orders } = await useAsyncData('orders', () => $fetch('/api/orders'))
```

## Server Routes (Nitro / H3)

File-based routing in `server/api/`. Method suffix in filename:

```ts
// server/api/users/[id].get.ts
export default defineEventHandler(async (event) => {
  const { id } = getRouterParams(event)
  const user = await useDB().query.users.findFirst({ where: eq(tables.users.id, id) })
  if (!user) throw createError({ statusCode: 404, message: 'User not found' })
  return user
})
```

```ts
// server/api/users/index.post.ts
export default defineEventHandler(async (event) => {
  const body = await useValidatedBody(event, z.object({  // h3-zod
    name: z.string().min(1),
    email: z.string().email(),
  }))
  return createUser(body)
})
```

Use `h3-zod` (`useValidatedBody`, `useValidatedParams`) for all server route validation.

## Authentication: nuxt-auth-utils

Session management for Nuxt apps. No config required beyond `nuxt-auth-utils` in modules:

```ts
// Server side (event handlers)
const session = await getUserSession(event)
await setUserSession(event, { user: { id, email, name } })
await clearUserSession(event)

// Protect server routes
const { user } = await requireUserSession(event)
```

```vue
<!-- Client side -->
<script setup lang="ts">
const { user, loggedIn, clear: logout } = useUserSession()
</script>

<template>
  <div v-if="loggedIn">Hello {{ user.name }}</div>
  <button v-else @click="navigateTo('/login')">Log in</button>
</template>
```

## NuxtHub (Cloudflare Workers + D1)

For apps deployed to Cloudflare Workers with edge-side D1 database:

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  modules: ['@nuxthub/core', '@nuxt/ui', 'nuxt-auth-utils'],
  hub: { database: true },
})
```

Access D1 database via the `useDB()` utility (see [database.md](database.md)):
```ts
// server/api/tasks/index.get.ts
export default defineEventHandler(async (event) => {
  const { user } = await requireUserSession(event)
  const tasks = await useDB().select().from(tables.tasks).where(eq(tables.tasks.userId, user.id))
  return tasks
})
```

Development with Wrangler:
```json
{ "dev": "nuxt dev --remote" }
```

## @nuxt/ui Components

Always use `@nuxt/ui` components in Nuxt 3 apps. Provides `U`-prefixed components built on Headless UI + Tailwind:

```vue
<template>
  <UContainer>
    <UCard>
      <template #header>
        <h2>Users</h2>
      </template>
      <UTable :rows="users" :columns="columns" />
      <UButton color="primary" @click="openModal">Add User</UButton>
    </UCard>

    <UModal v-model="isOpen">
      <UForm :schema="schema" :state="formState" @submit="onSubmit">
        <UFormGroup label="Name" name="name">
          <UInput v-model="formState.name" />
        </UFormGroup>
        <UButton type="submit">Save</UButton>
      </UForm>
    </UModal>
  </UContainer>
</template>
```

## Storyblok (Headless CMS)

Used in marketing and content-focused Nuxt sites:

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  modules: ['@storyblok/nuxt'],
  storyblok: { accessToken: process.env.STORYBLOK_TOKEN },
})
```

```vue
<script setup lang="ts">
const story = await useAsyncStoryblok('home', { version: 'published' })
</script>

<template>
  <StoryblokComponent v-if="story" :blok="story.content" />
</template>
```

## Inertia.js (AdonisJS + Vue)

For server-driven Vue pages rendered by AdonisJS controllers:

```ts
// AdonisJS controller
return inertia.render('Dashboard', { user, stats })
```

```vue
<!-- pages/Dashboard.vue — default export required for Inertia -->
<script setup lang="ts">
defineProps<{ user: User; stats: Stats }>()
</script>
```

## UI Libraries

| Project type | Library |
|---|---|
| Nuxt 3 apps | `@nuxt/ui` (always) |
| Marketing/content | `@nuxt/ui` + Storyblok |
| Legacy Vue apps | Naive UI |

## TypeScript in Vue / Nuxt

**Props with complex types**:
```ts
// Inline interface (preferred for simple props)
const props = defineProps<{
  userId: string
  status: 'active' | 'inactive'
  tags?: string[]
}>()

// Named interface for reuse
interface UserCardProps {
  user: User
  onEdit?: (id: string) => void
}
const props = defineProps<UserCardProps>()
```

**Emits with payload types**:
```ts
const emit = defineEmits<{
  update: [id: string, data: Partial<User>]
  delete: [id: string]
  close: []
}>()

emit('update', userId, { name: 'New Name' })
```

**Typed `ref` and `computed`**:
```ts
const count = ref<number>(0)
const user = ref<User | null>(null)
const items = ref<string[]>([])

// computed infers type automatically — only annotate when it helps
const fullName = computed(() => `${props.firstName} ${props.lastName}`)
const activeUsers = computed<User[]>(() => users.value.filter(u => u.active))
```

**Typed composable return**:
```ts
// composables/useAuth.ts
interface UseAuthReturn {
  user: Ref<User | null>
  isLoggedIn: ComputedRef<boolean>
  login: (email: string, password: string) => Promise<void>
  logout: () => Promise<void>
}

export function useAuth(): UseAuthReturn {
  const user = ref<User | null>(null)
  const isLoggedIn = computed(() => user.value !== null)
  // ...
  return { user, isLoggedIn, login, logout }
}
```

**Typed server routes (H3)**:
```ts
import type { EventHandler, H3Event } from 'h3'

// Define typed response shape
interface UserResponse {
  id: string
  name: string
  email: string
}

export default defineEventHandler(async (event: H3Event): Promise<UserResponse> => {
  const { id } = getRouterParams(event)
  const user = await findUser(id)
  if (!user) throw createError({ statusCode: 404, statusMessage: 'Not found' })
  return user
})
```

**`useAsyncData` typing**:
```ts
// Return type is inferred, but can be annotated for clarity
const { data: users, pending, error } = await useAsyncData<User[]>(
  'users',
  () => $fetch('/api/users'),
)
// users is Ref<User[] | null>
```

**`definePageMeta` and route typing**:
```ts
definePageMeta({
  middleware: ['auth'],
  layout: 'dashboard',
})
```

<!--
Source references:
- https://nuxt.com/docs
- https://ui.nuxt.com
- https://hub.nuxt.com/docs
- https://nuxt-auth-utils.vercel.app
- https://www.storyblok.com/docs/nuxt
-->
