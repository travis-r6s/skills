---
name: tooling
description: Travis's preferred toolchain — Biome, pnpm, Doppler, tsup, tsx, codegen. Use when setting up projects, writing scripts, or configuring linting/formatting.
---

# My Tooling

## Package Manager: pnpm

Always `pnpm`. Never npm or yarn in new projects.

```json
{
  "engines": { "node": ">=20.x" }
}
```

Run scripts with: `pnpm run <script>` or `pnpm <script>`.
Workspace packages: `pnpm-workspace.yaml`.

## Linting & Formatting: Biome

Biome replaces both ESLint and Prettier in all new/modern projects.

**Standard `biome.json` for apps (180 char line width):**
```json
{
  "$schema": "https://biomejs.dev/schemas/1.9.4/schema.json",
  "organizeImports": { "enabled": true },
  "linter": {
    "enabled": true,
    "ignore": ["dist", "node_modules", ".output", ".nuxt"],
    "rules": {
      "recommended": true,
      "style": { "useImportType": "error" }
    }
  },
  "formatter": {
    "enabled": true,
    "ignore": ["dist", "node_modules"],
    "indentWidth": 2,
    "indentStyle": "space",
    "lineWidth": 180
  },
  "javascript": {
    "formatter": {
      "quoteStyle": "single",
      "semicolons": "asNeeded",
      "trailingCommas": "all"
    }
  }
}
```

**For libraries (120 char line width):**
Same as above but `"lineWidth": 120`.

**Script names are always:**
```json
{
  "lint": "biome check --write .",
  "lint:unsafe": "biome check --write --unsafe ."
}
```

**ESLint is used in:** Adonis v6 projects (uses `@adonisjs/eslint-config`) and legacy projects. Prefer migrating to Biome when touching those projects.

## TypeScript Executor: tsx

For running `.ts` scripts directly:
```json
{ "start": "tsx src/index.ts" }
```

## Build Tool: tsdown

`tsdown` replaces `tsup` for building library packages. Powered by Rolldown — faster and more configurable.

**Scripts:**
```json
{
  "scripts": {
    "build": "tsdown --dts --sourcemap --clean",
    "dev": "tsdown --watch=./src"
  }
}
```

**`tsdown.config.ts`** for multi-entry or custom builds:
```ts
import { defineConfig } from 'tsdown'

// Single entry
export default defineConfig({
  entry: 'src/index.ts',
  dts: true,
  sourcemap: true,
  clean: true,
  format: ['esm'],
  target: ['node20', 'es2024'],
  platform: 'neutral',
})

// Multiple entries
export default defineConfig({
  entry: {
    index: 'src/index.ts',
    utils: 'src/utils/index.ts',
    types: 'src/types.ts',
  },
  dts: { sourcemap: true },
  clean: true,
  format: ['esm'],
})

// Unbundled (preserve module structure — good for tree-shaking)
export default defineConfig({
  entry: ['src/index.ts'],
  unbundle: true,
  platform: 'neutral',
})
```

Output goes to `dist/`. Install: `pnpm add -D tsdown`.

## Secrets: Doppler

Always use Doppler — never `.env` files in scripts or committed secrets.

**Always prefix dev/test scripts with `doppler run --`:**
```json
{
  "dev": "doppler run -- node ace serve --watch",
  "test": "doppler run --config dev_test -- node ace test",
  "start": "doppler run -- tsx src/index.ts"
}
```

**`doppler.yaml` in project root:**
```yaml
setup:
  - project: my-project
    config: dev
```

## TypeScript Config

**Baseline `tsconfig.json` for scripts/APIs:**
```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ES2022",
    "moduleResolution": "Node",
    "allowSyntheticDefaultImports": true,
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "strict": true,
    "skipLibCheck": true,
    "skipDefaultLibCheck": true
  }
}
```

For bundler-resolved projects (`Vite`, `electron-vite`): use `"moduleResolution": "Bundler"`.

## Database Migrations: Drizzle Kit

For Drizzle ORM projects (Cloudflare D1 / SQLite):

```json
{
  "db:generate": "drizzle-kit generate",
  "db:migrate": "drizzle-kit migrate"
}
```

For Kysely projects (PostgreSQL), use `node-pg-migrate`:
```json
{
  "db:migrate": "node-pg-migrate up"
}
```

## Code Generation

For REST APIs, generate types from OpenAPI spec:
```json
{
  "codegen:schemas": "openapi-typescript --enum --enum-values --dedupe-enums --root-types"
}
```

For GraphQL, use `graphql-codegen`:
```json
{
  "codegen:graphql": "graphql-codegen --config codegen.ts"
}
```

Run all codegen together:
```json
{
  "codegen": "pnpm run /^codegen:.*/"
}
```

## Standard Script Names

All projects follow this naming convention for npm scripts:

| Script | Purpose |
|---|---|
| `dev` | Local development server |
| `build` | Production build |
| `start` | Run production build |
| `lint` | Lint and format (auto-fix) |
| `typecheck` | `tsc --noEmit` |
| `test` | Run test suite |
| `codegen` | Run all code generators |

## Testing

- **Japa** in AdonisJS projects (`@japa/runner`, `@japa/preset-adonis`)
- **Vitest** in Vite-based projects
- Test files live in `/tests` directory (not co-located)
