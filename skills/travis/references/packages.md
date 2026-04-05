---
name: packages
description: Travis's preferred npm packages and how he uses them. Use when adding dependencies to a project, choosing between libraries, or writing code that uses these packages.
---

# My Preferred Packages

Always check if one of these is already in use before reaching for a new library. Prefer what's listed here.

## Dates: Luxon

**Always Luxon**. Never moment.js. Never date-fns.

```ts
import { DateTime } from 'luxon'

// Parse ISO
const date = DateTime.fromISO(isoString, { zone: 'America/New_York' })

// Navigation
const lastMonth = DateTime.now().startOf('month').minus({ months: 1 })
const startDate = date.startOf('month')
const endDate = date.endOf('month')

// Output
date.toISODate()        // '2024-01-15'
date.toISO({ suppressMilliseconds: true })
date.toFormat('dd MMM yyyy')

// Validation
if (date.invalidExplanation) { logger.error('Invalid date:', date.invalidExplanation) }
```

## Validation: Zod

```ts
import { z } from 'zod'

const schema = z.object({
  name: z.string().min(1),
  email: z.string().email(),
  amount: z.number().positive(),
})

type Schema = z.infer<typeof schema>
const parsed = schema.parse(rawInput)
const result = schema.safeParse(rawInput)
```

## Utilities: Radashi

Radashi replaces lodash in all new code.

```ts
import * as r from 'radashi'

// Parallel with concurrency limit
const results = await r.parallel({ limit: 4 }, items, async (item) => process(item))

// Group
const grouped = r.group(items, (item) => item.category)

// Unique
const unique = r.unique(items, (item) => item.id)

// Flat
const flat = r.flat(nestedArrays)
```

## ID Generation: ulidx

Always ULIDs for sortable, unique IDs. Never `uuid` in new code.

```ts
import { ulid } from 'ulidx'

const id = ulid()  // '01ARZ3NDEKTSV4RRFFQ69G5FAV'
```

## Money: currency.js

```ts
import currency from 'currency.js'

const price = currency(19.99)
const total = price.multiply(quantity).value
const formatted = currency(amount, { symbol: '£' }).format()
```

Dinero.js is used in older projects — prefer currency.js for new work.

## Deep Defaults: defu

```ts
import { defu } from 'defu'

const config = defu(userConfig, defaultConfig)
// Deeply merges, with userConfig taking precedence
```

## Logging: consola

Always consola for scripts and CLI tools. Never `console.log` in scripts.

```ts
import consola from 'consola'

// Tagged logger (preferred)
const logger = consola.withTag('My Script')

// Log levels
logger.info('Starting...')
logger.start('Fetching data...')
logger.success('Done!')
logger.warn('Skipping item...')
logger.error(error)
logger.box('Summary message')

// Interactive prompts
const value = await consola.prompt('Enter value:', {
  type: 'text',
  required: true,
  cancel: 'null',
  initial: 'default',
})

const selected = await logger.prompt('Select stores:', {
  type: 'multiselect',
  cancel: 'reject',
  required: true,
  options: items.map((item) => ({ label: item.name, value: item.id })),
})
```

## GraphQL Client: graphql-request

```ts
import { GraphQLClient, gql } from 'graphql-request'

const client = new GraphQLClient(endpoint, {
  headers: { Authorization: `Bearer ${token}` },
})

const GET_ORDERS = gql`
  query GetOrders($first: Int!) {
    orders(first: $first) { id name }
  }
`

const data = await client.request(GET_ORDERS, { first: 50 })
```

Use `graphql-codegen` to generate typed SDK from schema.

## Error Tracking: Sentry

```ts
import * as Sentry from '@sentry/node'

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.NODE_ENV,
  tracesSampleRate: process.env.NODE_ENV === 'production' ? 0.2 : 1,
  profilesSampleRate: process.env.NODE_ENV === 'production' ? 0.2 : 0,
})

// Breadcrumbs for context
Sentry.addBreadcrumb({ message: 'Started processing order' })

// Capture with context
Sentry.captureException(error, { extra: { orderId, shop } })
```

## Azure SDK

```ts
import { CosmosClient } from '@azure/cosmos'
import { BlobServiceClient } from '@azure/storage-blob'
import { app, input, output } from '@azure/functions'
```

Use `cosmos-orm` (custom package at `/home/travis/projects/wind/cosmos-orm`) for Cosmos DB:
```ts
import { createClient } from 'cosmos-orm'

const db = createClient({
  database: 'my-db',
  connectionStringSetting: 'COSMOS_CONNECTION_STRING',
  models: (builder) => ({
    shops: builder.createModel<Shop>('shops'),
    orders: builder.createModel<Order>('orders'),
  }),
})

const shop = await db.shops.find(shopId)
const allShops = await db.shops.all()
await db.shops.create({ id: ulid(), name: 'My Shop' })
```

## Shopify

```ts
import { parseGid } from '@shopify/admin-graphql-api-utilities'

const id = parseGid(globalId)  // Extract numeric ID from GID
```

Use `shopify-bulk-export` for bulk operations.

## String Utilities

```ts
import { camelCase, pascalCase, kebabCase } from 'change-case'
```

## Config Files: citty

For building CLI tools with subcommands:
```ts
import { defineCommand, runMain } from 'citty'

const main = defineCommand({
  meta: { name: 'my-tool', description: 'Does things' },
  args: { input: { type: 'string', required: true } },
  run: async ({ args }) => { },
})

runMain(main)
```

## Email: React Email

```tsx
import { render } from '@react-email/render'
import { Html, Body, Text } from '@react-email/components'

export function WelcomeEmail({ name }: { name: string }) {
  return (
    <Html>
      <Body>
        <Text>Hello {name}</Text>
      </Body>
    </Html>
  )
}

const html = await render(<WelcomeEmail name="Travis" />)
```

## State Management: Zustand

For React apps that need persistent client state:

```ts
import { create } from 'zustand'
import { persist } from 'zustand/middleware'

interface AuthStore {
  user: User | null
  setUser: (user: User | null) => void
}

export const useAuthStore = create<AuthStore>()(
  persist(
    (set) => ({
      user: null,
      setUser: (user) => set({ user }),
    }),
    { name: 'auth-store' },
  ),
)
```

## Server State: TanStack React Query

For data fetching and caching in React/Next.js:

```ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query'

function useUser(id: string) {
  return useQuery({
    queryKey: ['user', id],
    queryFn: () => fetchUser(id),
  })
}

function useUpdateUser() {
  const queryClient = useQueryClient()
  return useMutation({
    mutationFn: updateUser,
    onSuccess: (_, { id }) => queryClient.invalidateQueries({ queryKey: ['user', id] }),
  })
}
```

Pair with `graphql-request` for typed GraphQL queries in Next.js apps.

## Nuxt Auth: nuxt-auth-utils

For session management in Nuxt 3 apps. Provides server-side session helpers and `useUserSession` composable:

```ts
// In Nuxt server routes
const session = await getUserSession(event)
await setUserSession(event, { user: { id, email } })
await clearUserSession(event)

// In Nuxt pages/composables
const { user, loggedIn, clear: logout } = useUserSession()
```

## H3/Nitro Request Validation: h3-zod

For validating request bodies and params in Nuxt server routes:

```ts
import { useValidatedBody, z } from 'h3-zod'

export default defineEventHandler(async (event) => {
  const { name, email } = await useValidatedBody(event, z.object({
    name: z.string().min(1),
    email: z.string().email(),
  }))

  return createUser({ name, email })
})
```

## Headless CMS

**Storyblok** — for marketing/content sites:
```ts
import { useStoryblok } from '@storyblok/js'
```

**Sanity** — for complex structured content:
```ts
import { createClient } from '@sanity/client'

const client = createClient({
  projectId: process.env.SANITY_PROJECT_ID,
  dataset: 'production',
  useCdn: true,
  apiVersion: '2024-01-01',
})
```

## What to Avoid

| Avoid | Use instead |
|---|---|
| `moment` | `luxon` |
| `date-fns` | `luxon` |
| `lodash` | `radashi` |
| `uuid` | `ulidx` |
| `axios` | native `fetch` or `graphql-request` |
| `winston` / `pino` | `consola` (for scripts) |
| `dotenv` | Doppler |
