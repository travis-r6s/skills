---
name: travis
description: Travis Reynolds's personal coding conventions, architecture patterns, and tooling preferences for TypeScript/JavaScript projects. Use when writing any code in Travis's projects, setting up new projects, or making architectural decisions.
metadata:
  author: Travis Reynolds
  version: "2026.04.05"
---

## Coding Style

Travis uses strict TypeScript with Biome for formatting. Key rules:
- 2 spaces, single quotes, no semicolons, trailing commas, 120–180 char line width
- `import type` for all type-only imports (Biome enforces this as an error)
- Named imports over defaults (except Adonis IoC and Vue/React components)
- `private` keyword — never underscore prefixes

For full style rules: [coding-style](references/coding-style.md)

---

## Architecture

Business logic lives in service classes with explicit public methods. Stateless utilities are plain exported functions. Multi-step workflows use numbered folders with typed orchestrators.

```ts
// Services — app/Services/TrackingService.ts
export default class TrackingService {
  public async createTrackings(fulfillments: Fulfillment[], shopifyOrderId: string): Promise<void> { }
  private async processFulfillment(fulfillment: Fulfillment, orderId: string): Promise<void> { }
}

// Utilities — utils/links.ts (named exports, no classes)
export const createCustomOrderAppLink = (id: string, shop: Shop): string => { }
```

For full architecture patterns: [architecture](references/architecture.md)

---

## Async & Error Handling

Always `async/await` — never `.then()` chains. `radashi r.parallel` for controlled concurrency. Human-readable error messages with `{ cause }` chaining. Always initialise Sentry early.

```ts
const results = await r.parallel({ limit: 4 }, stores, async (store) => processStore(store))
throw new Error('Failed to create charge', { cause: originalError })
```

For full patterns: [async-and-errors](references/async-and-errors.md)

---

## Tooling

| Tool | Purpose |
|---|---|
| `pnpm` | Always — never npm or yarn |
| `biome` | Lint + format — replaces ESLint/Prettier |
| `tsx` | Run `.ts` scripts directly |
| `tsup` | Build library packages |
| `doppler run --` | Prefix all dev/test scripts — never `.env` files |

For config templates and full toolchain: [tooling](references/tooling.md)

---

## Preferred Packages

| Category | Package |
|---|---|
| Dates | `luxon` (never moment/date-fns) |
| Validation | `zod` |
| Utilities | `radashi` (replaces lodash) |
| IDs | `ulidx` (ULIDs) |
| Money | `currency.js` |
| Logging | `consola` with `.withTag()` |
| Config merging | `defu` |
| GraphQL client | `graphql-request` |
| Error tracking | `@sentry/node` |

For usage examples: [packages](references/packages.md)

---

## New Projects

Always `pnpm init`. Always Biome + TypeScript strict + Doppler. `"type": "module"` in package.json. Standard script names: `dev`, `build`, `start`, `lint`, `typecheck`, `test`, `codegen`.

Full checklist: [new-project](references/new-project.md)

---

## References

| Topic | Description | Reference |
|-------|-------------|-----------|
| Coding Style | Formatting, naming conventions, TypeScript rules, imports | [coding-style](references/coding-style.md) |
| Architecture | Services, utilities, orchestrators, AdonisJS patterns | [architecture](references/architecture.md) |
| Async & Errors | async/await patterns, concurrency, error handling, Sentry | [async-and-errors](references/async-and-errors.md) |
| Tooling | Biome, pnpm, Doppler, tsup, tsx | [tooling](references/tooling.md) |
| New Project | Project setup checklist and conventions | [new-project](references/new-project.md) |
| Packages | Preferred npm packages by category | [packages](references/packages.md) |
| Frontend | React/Vue/Nuxt component patterns, Tailwind | [frontend](references/frontend.md) |
| GraphQL | Pothos schema builder, graphql-request, codegen | [graphql](references/graphql.md) |
