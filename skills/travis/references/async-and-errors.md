---
name: async-and-errors
description: Travis's patterns for async code, error handling, and Sentry integration. Use when writing any async functions, error boundaries, or adding observability.
type: skill
---

# Async & Error Handling Patterns

## Always async/await

Never raw promise chains. Always `async/await`.

```ts
// ✅
const { resources } = await this.client.items.readAll().fetchAll()
return resources as CosmosItemDefinition<T>[]

// ❌ — no .then() chains
this.client.items.readAll().fetchAll().then(({ resources }) => resources)
```

## Concurrency

**Use `radashi r.parallel` for controlled concurrency in new code:**
```ts
import * as r from 'radashi'

const results = await r.parallel({ limit: 4 }, stores, async (store) => {
  return processStore(store)
})
```

**Use `Promise.all` for simple fan-out:**
```ts
await Promise.all(
  fulfillments.map(async (fulfillment) => {
    await Promise.all(
      fulfillment.lineItems.map(async (lineItem) => processLineItem(lineItem)),
    )
  }),
)
```

Never use `Promise.allSettled` unless you specifically need to collect all failures.

## Error Messages

Human-readable messages. Use `{ cause: error }` for chaining:

```ts
// Simple
throw new Error(`No store found with ID: ${storeId}`)

// With original error chained
throw new Error('Failed to create payment charge', { cause: originalError })

// With guard
if (!store) throw new Error(`No store found with that ID.`)
```

Never swallow errors silently. Always either throw, log, or propagate.

## Try/Catch Pattern

Keep try blocks narrow — only around the operation that can fail:

```ts
// ✅ Narrow try
let result: OrderData
try {
  result = await fetchOrderFromShopify(orderId)
} catch (error) {
  throw new Error(`Failed to fetch order ${orderId}`, { cause: error })
}
// Use result here

// ❌ Too wide
try {
  const result = await fetchOrderFromShopify(orderId)
  const processed = processOrder(result)
  await saveToDatabase(processed)
} catch (error) {
  // Which operation failed?
}
```

## Sentry Integration

Always initialise Sentry early (before any async work):

```ts
import * as Sentry from '@sentry/node'

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.NODE_ENV,
  release: process.env.npm_package_version,
  tracesSampleRate: process.env.NODE_ENV === 'production' ? 0.2 : 1,
  profilesSampleRate: process.env.NODE_ENV === 'production' ? 0.2 : 0,
})
```

Use breadcrumbs to trace orchestration steps:
```ts
Sentry.addBreadcrumb({ message: `Started export orchestrator with ID: ${input.exportId}` })
Sentry.addBreadcrumb({ message: 'Fetching stores...' })
// ... do work ...
Sentry.addBreadcrumb({ message: 'Generating archive' })
```

Capture with extra context:
```ts
Sentry.captureException(error, {
  extra: { orderId, shop: shop.id, exportId },
})
```

**Azure Functions wrapper pattern:**
```ts
export async function captureFuncException(
  context: InvocationContext,
  error: unknown,
  extra?: Record<string, unknown>,
): Promise<void> {
  Sentry.captureException(error, {
    extra: { functionName: context.functionName, invocationId: context.invocationId, ...extra },
  })
  await Sentry.flush(2000)
}
```

## Validation Before Proceeding

Use early returns and guards rather than nested conditionals:

```ts
// ✅ Guard early
const shop = await db.shops.find(shopId)
if (!shop) throw new Error(`Shop not found: ${shopId}`)

const config = shop.config
if (!config) throw new Error(`No config for shop: ${shopId}`)

// Continue with validated data
```

## Logging During Async Workflows

Use consola tags with step-by-step logging for long scripts:

```ts
const logger = consola.withTag('Export')

logger.start('Fetching orders...')
const orders = await fetchOrders(dateRange)
logger.info(`Fetched ${orders.length} orders`)

logger.start('Processing...')
const results = await r.parallel({ limit: 4 }, orders, processOrder)
logger.success(`Processed ${results.length} orders`)
```

## Type-Safe Error Handling

In catch blocks, type the error as `unknown` (TypeScript strict mode enforces this):

```ts
try {
  await doWork()
} catch (error) {
  if (error instanceof Error) {
    logger.error('Failed:', error.message)
    throw new Error('Context-specific message', { cause: error })
  }
  throw error
}
```
