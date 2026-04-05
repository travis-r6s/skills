---
name: architecture
description: Travis's architectural patterns — services, orchestrators, numbered workflow steps, AdonisJS structure, Azure Functions. Use when designing new features, adding modules, or structuring new projects.
---

# My Architecture Patterns

## Service Classes

Business logic lives in service classes. Services have clear public methods and private helpers.

```ts
// app/Services/TrackingService.ts
export default class TrackingService {
  public async createTrackings(fulfillments: Fulfillment[], shopifyOrderId: string): Promise<void> {
    await Promise.all(
      fulfillments.map(async (fulfillment) => {
        await this.processFulfillment(fulfillment, shopifyOrderId)
      }),
    )
  }

  private async processFulfillment(fulfillment: Fulfillment, orderId: string): Promise<void> { }

  private formatTrackingCompany(company: string | null): string {
    return company?.toLowerCase() ?? 'unknown'
  }
}
```

Rules:
- `private` keyword — never underscore prefix
- Explicit return types on public methods
- Private helpers do one thing
- No static methods unless truly stateless

## Pure Utility Modules

Stateless utilities are plain exported functions, not classes:

```ts
// utils/links.ts
export const createCustomOrderAppLink = (customOrderId: string, shop: Shop): string => {
  return `https://${shop.domain}/admin/orders/${customOrderId}`
}

export async function formatDateKey(orderDate: DateTime): Promise<string> {
  return orderDate.toFormat('yyyy-MM')
}
```

Rules:
- Named exports only (no default export)
- No classes for utility modules
- Pure functions where possible

## Orchestrator Pattern (Workflow Scripts)

Multi-step workflows are split into numbered folders with orchestrators:

```
src/
  1.generator/
    orchestrator.ts       ← accepts typed Input, calls activities
    activities/
      1.fetch-orders.ts
      2.generate-archive.ts
  2.exporter/
    orchestrator.ts
    activities/
      1.prepare-data.ts
      2.upload-to-storage.ts
      3.send-email.ts
  utils/
  types/
  index.ts               ← top-level entry, interactive prompts
```

**Orchestrator signature:**
```ts
export interface Input {
  /** A unique ID for this export. */
  exportId: string
  /** The stores to run this export for. */
  stores: SchemaDbShop[]
  /** ISO date range. @example { start: '2024-01-01', end: '2024-06-01' } */
  exportDateRange: { start: string; end: string }
}

export const Name = 'RoyaltyExportsGenerator'

export async function orchestrator(input: Input) {
  Sentry.addBreadcrumb({ message: `Started with ID: ${input.exportId}` })

  const results = await r.parallel({ limit: 4 }, input.stores, async (store) =>
    SubOrchestrator.orchestrator({ shop: store, ...input }),
  )

  return { id: input.exportId, results }
}
```

**Index entry (interactive):**
```ts
import consola from 'consola'
import { ulid } from 'ulidx'
import * as GeneratorOrchestrator from './1.generator/orchestrator'

const logger = consola.withTag('My Script')

logger.info('Starting script')
const startDate = await consola.prompt('Enter start date:', { type: 'text', required: true })
if (!startDate) { logger.warn('Cancelled.'); process.exit() }

const result = await GeneratorOrchestrator.orchestrator({
  exportId: ulid(),
  exportDateRange: { start: startDate, end: endDate },
  stores: selectedStores,
})

logger.success(`Done: ${result.id}`)
```

## DB Client Pattern (Singletons)

Database connections are exported as singleton objects, not instantiated per-use:

```ts
// utils/cosmos.ts
import { createClient } from 'cosmos-orm'

export const exportsDB = createClient({
  database: 'exports',
  connectionStringSetting: 'COSMOS_CONNECTION_STRING',
  models: (builder) => ({
    shops: builder.createModel<Shop>('shops'),
    orders: builder.createModel<Order>('orders'),
    exports: builder.createModel<Export>('exports'),
  }),
})
```

Import and use directly:
```ts
import { exportsDB } from '@/utils/cosmos'

const shops = await exportsDB.shops.all()
const shop = await exportsDB.shops.find(shopId)
```

## AdonisJS Project Structure (v5 and v6)

```
app/
  Controllers/        ← HTTP request handlers (thin, delegate to services)
  Services/           ← Business logic
  Models/             ← Lucid ORM models
  Mailers/            ← Email templates
  GraphQL/
    Types/            ← Pothos type definitions
    builder.ts        ← Pothos builder setup
    schema.ts         ← Schema assembly
  Middleware/
  Policies/           ← Bouncer authorization
  Exceptions/
config/
database/
  migrations/
  seeders/
start/
  routes.ts
tests/
```

**AdonisJS v5 IoC imports:**
```ts
import Logger from '@ioc:Adonis/Core/Logger'
import Database from '@ioc:Adonis/Lucid/Database'
import { BaseModel, column, hasOne } from '@ioc:Adonis/Lucid/Orm'
```

**AdonisJS v6 package imports (with `#` aliases):**
```ts
import { BaseModel, column } from '@adonisjs/lucid/orm'
// Aliases in package.json "imports":
import User from '#models/user'
import UserService from '#services/user_service'
```

**Lucid Model pattern:**
```ts
// app/Models/User.ts
export default class User extends BaseModel {
  public static table = 'users'

  @column({ isPrimary: true })
  public id: number

  @column()
  public email: string

  @column.dateTime({ autoCreate: true })
  public createdAt: DateTime

  @column.dateTime({ autoCreate: true, autoUpdate: true })
  public updatedAt: DateTime
}
```

## Azure Functions Structure

```
src/
  functions/
    my-function/
      index.ts       ← Azure Function registration
      handler.ts     ← Business logic
  utils/
    cosmos.ts        ← DB client
    sentry.ts        ← Error tracking setup
  types/
  index.ts
```

**Function registration:**
```ts
import { app } from '@azure/functions'
import { handler } from './handler'

app.http('my-function', {
  methods: ['POST'],
  authLevel: 'function',
  route: 'my-route',
  handler,
})
```

## Nuxt 3 / Vue Structure

```
pages/
server/
  api/
  middleware/
components/
composables/
utils/
app.vue
nuxt.config.ts
```

Use `@nuxt/ui` for UI components. Use `Kysely` for database queries. Use `@xata.io/client` for Xata.

## Electron App Structure (electron-vite)

```
src/
  main/         ← Node.js main process
  preload/      ← Preload scripts
  renderer/     ← Vue/React frontend
  shared/       ← Shared types and utils
```

## Project Naming

- Repository/package names: `kebab-case`
- Monorepo workspace packages: grouped by type (`apis/`, `apps/`, `scripts/`, `themes/`)
- Script versions: `v1/`, `v2/`, `v3/` directories for generational improvements
