---
name: coding-style
description: Travis's personal code style rules — formatting, naming, TypeScript strictness, import conventions. Use when writing any new code in Travis's projects.
type: skill
---

# My Coding Style

When writing code in my projects, follow these rules exactly. Do not deviate.

## Formatting Rules

- **2 spaces** — never tabs
- **Single quotes** — `'string'` not `"string"` (JSON files use double quotes)
- **No semicolons** — Biome `asNeeded`
- **Trailing commas everywhere** on multiline — `trailingCommas: "all"`
- **Line width 120–180 chars** — 120 for libraries/packages, 180 for app code
- **Spaces inside braces**: `{ key: value }` not `{key: value}`

## Naming

| Context | Style | Example |
|---|---|---|
| Utility/config files | kebab-case | `url-builder.ts`, `cosmos-client.ts` |
| Class/component files | PascalCase | `BaseModel.ts`, `UserService.ts` |
| Adonis MVC directories | PascalCase | `Services/`, `Controllers/`, `Models/` |
| Variables & functions | camelCase | `createOrderLink`, `formatDateKey` |
| Interfaces | PascalCase | `ModelOptions`, `ExportInput` |
| Types | PascalCase | `CosmosResource<T>` |
| Enums | PascalCase + SCREAMING_SNAKE values | `enum Status { ACTIVE = 'ACTIVE' }` |
| React/Vue components | PascalCase (file + function/const) | `DataTable.tsx` |

## TypeScript Strictness

Always `"strict": true`. Always `ES2022` target. Always `skipLibCheck: true`.

```ts
// ✅ Explicit return types on public/exported functions
export async function fetchUser(id: string): Promise<User | undefined> { }

// ✅ Type imports separated with `import type`
import type { ModelOptions } from './base-model'
import { BaseModel } from './base-model'

// ✅ Interfaces for object shapes, types for unions/intersections/aliases
interface ModelOptions { database: string; container: string }
type CosmosResource<T extends Base> = Resource & T
type Base = object

// ✅ Constrained generics
export class BaseModel<T extends Base = typeof initial> { }

// ✅ Optional chaining + nullish coalescing
const conn = options.connectionString ?? process.env[setting]
const lower = company?.toLowerCase()

// ❌ Never any — use generics or unknown
// ❌ Never var
// ❌ Never CommonJS require()
```

## Import Order

Biome `organizeImports` is enabled. Imports flow: third-party → local, alphabetically within each group.

```ts
// Third party first
import { CosmosClient } from '@azure/cosmos'
import type { Container, Resource } from '@azure/cosmos'
import { ulid } from 'ulidx'

// Local imports after
import type { Base, ModelOptions } from './base-model'
import { formatDate } from './utils'
```

Named imports always over default imports — except:
- Adonis IoC default exports (`export default class TrackingService`)
- Vue/React components (`export default defineComponent(...)`)

## Private Members

Use the `private` keyword — never underscore prefix:
```ts
// ✅
private formatTrackingCompany(company: string): string { }

// ❌
_formatTrackingCompany(company: string): string { }
```

## JSDoc for Public APIs

All public methods and exported functions get JSDoc with `@example` blocks:
```ts
/**
 * Find a document by its ID.
 * @example
 * ```ts
 * const user = await db.users.find('user-123')
 * ```
 */
public async find(id: string): Promise<CosmosResource<T> | undefined> { }
```

## Biome Suppress Comments

When suppressing a Biome rule, use the standard comment with an explanation:
```ts
// biome-ignore lint/suspicious/noExplicitAny: <explanation>
```
