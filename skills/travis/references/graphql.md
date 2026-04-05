---
name: graphql
description: Travis's GraphQL patterns — Pothos schema builder, graphql-request client, codegen. Use when building or extending GraphQL APIs, adding resolvers, or consuming GraphQL endpoints.
---

# GraphQL Patterns

## Schema Building: Pothos

All GraphQL schemas are built with Pothos. Never write SDL (schema definition language) manually.

### Builder Setup

```ts
// app/GraphQL/builder.ts
import SchemaBuilder from '@pothos/core'
import DataloaderPlugin from '@pothos/plugin-dataloader'
import RelayPlugin from '@pothos/plugin-relay'
import ScopeAuthPlugin from '@pothos/plugin-scope-auth'
import ValidationPlugin from '@pothos/plugin-validation'
import type { ContextData } from './types'

export const builder = new SchemaBuilder<{
  Context: ContextData
  AuthScopes: { authenticated: boolean; admin: boolean }
}>({
  plugins: [ScopeAuthPlugin, ValidationPlugin, DataloaderPlugin, RelayPlugin],
  authScopes: async (ctx) => ({
    authenticated: !!ctx.user,
    admin: ctx.user?.role === 'admin',
  }),
  relay: { idFieldName: 'id' },
  validationOptions: { validationError: (error) => error },
})
```

### Object Types

```ts
builder.objectType(User, {
  name: 'User',
  fields: (t) => ({
    id: t.exposeID('id'),
    email: t.exposeString('email'),
    createdAt: t.string({
      resolve: (user) => user.createdAt.toISO()!,
    }),
  }),
})
```

### Query Fields

```ts
builder.queryField('user', (t) =>
  t.field({
    type: User,
    nullable: true,
    authScopes: { authenticated: true },
    args: { id: t.arg.id({ required: true }) },
    resolve: async (_, { id }, ctx) => {
      return ctx.services.user.find(id)
    },
  }),
)
```

### Mutation Fields with Input

```ts
builder.mutationField('createOrder', (t) =>
  t.fieldWithInput({
    type: Order,
    authScopes: { authenticated: true },
    input: {
      shopId: t.input.id({ required: true }),
      amount: t.input.int({ required: true }),
      notes: t.input.string(),
    },
    resolve: async (_, { input }, ctx) => {
      return ctx.services.order.create(input)
    },
  }),
)
```

### Dataloader Pattern

```ts
// app/GraphQL/dataloaders.ts
export const userLoader = createDataLoader(async (ids: readonly string[]) => {
  const users = await db.users.findMany(ids)
  return ids.map((id) => users.find((u) => u.id === id) ?? null)
})
```

### GraphQL Envelop Plugins

Standard plugin setup for AdonisJS + Yoga:
```ts
import { envelop } from '@envelop/core'
import { useResponseCache } from '@graphql-yoga/plugin-response-cache'
import { useSentry } from '@envelop/sentry'

const getEnveloped = envelop({
  plugins: [
    useSchema(schema),
    useSentry(),
    useResponseCache({ session: (ctx) => ctx.user?.id ?? null }),
  ],
})
```

## GraphQL Client: graphql-request

For consuming external GraphQL APIs:

```ts
import { GraphQLClient, gql } from 'graphql-request'

const client = new GraphQLClient('https://api.example.com/graphql', {
  headers: {
    'X-Shopify-Access-Token': token,
    'Content-Type': 'application/json',
  },
})

const ORDERS_QUERY = gql`
  query GetOrders($first: Int!, $after: String) {
    orders(first: $first, after: $after) {
      nodes { id name totalPrice }
      pageInfo { hasNextPage endCursor }
    }
  }
`

const data = await client.request<OrdersQuery>(ORDERS_QUERY, { first: 250 })
```

## Code Generation

Always generate types from the schema — never write GraphQL types by hand.

**`codegen.ts`:**
```ts
import type { CodegenConfig } from '@graphql-codegen/cli'

const config: CodegenConfig = {
  schema: 'https://api.example.com/graphql',
  documents: ['src/**/*.graphql', 'src/**/*.ts'],
  generates: {
    'src/types/graphql.ts': {
      plugins: ['typescript', 'typescript-operations'],
    },
    'src/utils/graphql-client.ts': {
      plugins: ['typescript-graphql-request'],
    },
  },
}

export default config
```

Run: `pnpm codegen:graphql`

For REST APIs — use `openapi-typescript`:
```bash
openapi-typescript https://api.example.com/openapi.json --enum --enum-values --dedupe-enums --root-types -o src/types/schema.ts
```

Import the generated types:
```ts
import type { components } from '@/types/schema'

type Order = components['schemas']['Order']
type OrderStatus = components['schemas']['OrderStatus']
```

## Pagination Pattern (Relay / Cursor-based)

```ts
builder.queryField('orders', (t) =>
  t.connection({
    type: Order,
    authScopes: { authenticated: true },
    resolve: async (_, args, ctx) => {
      return resolveOffsetConnection({ args }, async ({ offset, limit }) => {
        return ctx.services.order.list({ offset, limit })
      })
    },
  }),
)
```

## Schema Assembly

```ts
// app/GraphQL/schema.ts
import './Types/user'
import './Types/order'
import { builder } from './builder'

// Build queries and mutations in each Type file
export const schema = builder.toSchema()
```
