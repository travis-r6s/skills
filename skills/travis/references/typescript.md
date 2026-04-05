---
name: typescript
description: Travis's TypeScript patterns — utility types, discriminated unions, type guards, generics, Zod inference, module augmentation. Use when writing TypeScript types, generics, or narrowing logic in Node/server code.
---

# TypeScript Patterns (Node / Server)

Assumes `"strict": true`, `ES2022` target, `skipLibCheck: true`. See [coding-style.md](coding-style.md) for base rules (`import type`, interfaces vs types, naming).

## Utility Types

Prefer built-in utility types over manual type reconstruction:

```ts
// Extract a subset of fields
type UserPreview = Pick<User, 'id' | 'name' | 'email'>

// Omit sensitive fields
type PublicUser = Omit<User, 'passwordHash' | 'internalNotes'>

// Make fields optional for update payloads
type UserUpdate = Partial<Pick<User, 'name' | 'email' | 'avatarUrl'>>

// Require fields that are optional on the base type
type RequiredConfig = Required<Pick<Config, 'apiKey' | 'baseUrl'>>

// Unwrap a Promise return type
type OrderData = Awaited<ReturnType<typeof fetchOrder>>

// Extract parameter types
type FetchParams = Parameters<typeof fetchOrder>[0]

// Non-nullable
type UserId = NonNullable<User['id']>
```

## Discriminated Unions

Use a `type` or `kind` literal field to discriminate:

```ts
type WebhookEvent =
  | { type: 'order.created'; order: Order }
  | { type: 'order.cancelled'; orderId: string; reason: string }
  | { type: 'payment.failed'; paymentId: string; error: string }

function handleEvent(event: WebhookEvent) {
  switch (event.type) {
    case 'order.created':
      return processNewOrder(event.order)      // event.order is typed
    case 'order.cancelled':
      return cancelOrder(event.orderId)        // event.orderId is typed
    case 'payment.failed':
      return alertPaymentFailure(event.error)  // event.error is typed
  }
}
```

Result pattern:
```ts
type Result<T> =
  | { success: true; data: T }
  | { success: false; error: string }

function parseOrder(raw: unknown): Result<Order> {
  const parsed = orderSchema.safeParse(raw)
  if (!parsed.success) return { success: false, error: parsed.error.message }
  return { success: true, data: parsed.data }
}
```

## Type Guards

```ts
// `is` narrowing for unknown/union types
function isError(value: unknown): value is Error {
  return value instanceof Error
}

function isOrderEvent(event: WebhookEvent): event is Extract<WebhookEvent, { type: 'order.created' }> {
  return event.type === 'order.created'
}

// Usage
if (isError(caught)) {
  logger.error(caught.message)
}

// Assertion function (throws if wrong)
function assertDefined<T>(value: T | null | undefined, label: string): asserts value is T {
  if (value == null) throw new Error(`Expected ${label} to be defined`)
}

assertDefined(shop, 'shop')
// shop is now T (not T | null | undefined)
```

## Generics

Constrain generics — never leave them unconstrained unless truly necessary:

```ts
// Constrained to object shapes
export class Repository<T extends { id: string }> {
  async find(id: string): Promise<T | undefined> { }
  async create(data: Omit<T, 'id'>): Promise<T> { }
}

// Generic with default
export function createStore<T extends object = Record<string, unknown>>(initial: T) { }

// Infer return from argument
function groupBy<T, K extends string>(items: T[], key: (item: T) => K): Record<K, T[]> {
  return items.reduce((acc, item) => {
    const k = key(item)
    ;(acc[k] ??= []).push(item)
    return acc
  }, {} as Record<K, T[]>)
}
```

## `satisfies` Operator

Validate a value matches a type without widening it:

```ts
const config = {
  apiUrl: 'https://api.example.com',
  timeout: 30_000,
  retries: 3,
} satisfies Partial<AppConfig>
// config.apiUrl is still typed as 'https://api.example.com', not string
```

## `as const` Assertions

Lock array/object types to their literal values:

```ts
const STATUSES = ['active', 'inactive', 'pending'] as const
type Status = (typeof STATUSES)[number]  // 'active' | 'inactive' | 'pending'

const ROUTES = {
  home: '/',
  orders: '/orders',
  settings: '/settings',
} as const
type Route = (typeof ROUTES)[keyof typeof ROUTES]  // '/' | '/orders' | '/settings'
```

## Zod Type Inference

Always derive TypeScript types from Zod schemas — never duplicate type definitions:

```ts
import { z } from 'zod'

const CreateOrderSchema = z.object({
  shopId: z.string().ulid(),
  items: z.array(z.object({
    productId: z.string(),
    quantity: z.number().int().positive(),
  })),
  notes: z.string().optional(),
})

type CreateOrderInput = z.infer<typeof CreateOrderSchema>
// No separate interface needed

// Validate and narrow in one step
const input = CreateOrderSchema.parse(rawBody)
// input is now CreateOrderInput with full type safety
```

## Template Literal Types

```ts
type EventName = `${string}.created` | `${string}.updated` | `${string}.deleted`
type HttpMethod = 'GET' | 'POST' | 'PUT' | 'DELETE' | 'PATCH'
type ApiRoute = `/${string}`

// Combine with mapped types
type EventHandlers<T extends string> = {
  [K in T as `on${Capitalize<K>}`]: () => void
}
```

## Module Augmentation

For extending third-party types (e.g. AdonisJS HTTP context):

```ts
// types/adonis.d.ts
declare module '@ioc:Adonis/Core/HttpContext' {
  interface HttpContextContract {
    user: User | null
    shopId: string | null
  }
}

// Or for v6:
declare module '@adonisjs/core/http' {
  interface HttpContext {
    user: User | null
  }
}
```

## Mapped & Conditional Types

```ts
// Make all fields nullable
type Nullable<T> = { [K in keyof T]: T[K] | null }

// Deep partial
type DeepPartial<T> = { [K in keyof T]?: T[K] extends object ? DeepPartial<T[K]> : T[K] }

// Conditional: only string keys
type StringValues<T> = { [K in keyof T]: T[K] extends string ? K : never }[keyof T]
```

## Catch Blocks

TypeScript strict mode types caught errors as `unknown`. Always narrow:

```ts
try {
  await doWork()
} catch (error) {
  if (error instanceof Error) {
    throw new Error('Failed to process order', { cause: error })
  }
  throw error
}
```

<!--
Source references:
- https://www.typescriptlang.org/docs/handbook/utility-types.html
- https://www.typescriptlang.org/docs/handbook/2/narrowing.html
- https://www.typescriptlang.org/docs/handbook/2/generics.html
-->
