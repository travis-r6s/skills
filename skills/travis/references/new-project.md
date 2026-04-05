---
name: new-project
description: Travis's checklist and setup process for starting a new project. Use when initialising any new package, app, script, or service.
---

# New Project Setup

Follow this checklist when creating a new project from scratch.

## Package Manager

Always `pnpm`. Init with:
```bash
pnpm init
```

## package.json Baseline

```json
{
  "name": "my-project",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "engines": { "node": ">=20.x" },
  "scripts": {
    "dev": "doppler run -- tsx src/index.ts",
    "build": "tsdown --dts --sourcemap --clean",
    "lint": "biome check --write .",
    "typecheck": "tsc --noEmit",
    "start": "doppler run -- node dist/index.js"
  }
}
```

For library packages, add:
```json
{
  "main": "./dist/index.mjs",
  "module": "./dist/index.mjs",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "import": { "types": "./dist/index.d.mts", "default": "./dist/index.mjs" },
      "require": { "types": "./dist/index.d.ts", "default": "./dist/index.js" }
    }
  },
  "files": ["./dist/**/*"]
}
```

## TypeScript Config

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
    "skipDefaultLibCheck": true,
    "baseUrl": ".",
    "paths": { "@/*": ["./src/*"] }
  },
  "include": ["src"],
  "exclude": ["node_modules", "dist"]
}
```

For `Bundler` resolution (Vite, Nuxt, electron-vite), change `"moduleResolution": "Bundler"`.

## Biome Config

```json
{
  "$schema": "https://biomejs.dev/schemas/1.9.4/schema.json",
  "organizeImports": { "enabled": true },
  "linter": {
    "enabled": true,
    "ignore": ["dist", "node_modules"],
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

Use `"lineWidth": 120` for library packages.

## Doppler Config

```yaml
# doppler.yaml
setup:
  - project: my-project
    config: dev
```

Run: `doppler setup` to link the project.

## Dev Dependencies to Always Install

```bash
pnpm add -D @biomejs/biome typescript @types/node tsx tsdown
```

## Standard Directory Structure

**API / Script:**
```
src/
  index.ts
  types/
  utils/
  services/       (if applicable)
dist/
biome.json
tsconfig.json
doppler.yaml
package.json
pnpm-lock.yaml
```

**Workflow Script:**
```
src/
  1.step-one/
    orchestrator.ts
    activities/
  2.step-two/
    orchestrator.ts
    activities/
  utils/
  types/
  index.ts
```

**AdonisJS App (v6):**
```
app/
  controllers/
  models/
  services/
  middleware/
  validators/
  mails/
  policies/
  events/
  listeners/
database/
  migrations/
  seeders/
config/
start/
  routes.ts
  kernel.ts
tests/
resources/
public/
```

## Gitignore Essentials

```
node_modules/
dist/
.output/
.nuxt/
.env
*.env.local
```

## Common First Packages

For scripts:
```bash
pnpm add luxon ulidx consola radashi
pnpm add -D @types/luxon
```

For APIs:
```bash
pnpm add luxon ulidx zod @sentry/node consola radashi defu
pnpm add -D @types/luxon
```

For GraphQL APIs:
```bash
pnpm add graphql graphql-request
pnpm add -D @graphql-codegen/cli @graphql-codegen/typescript @graphql-codegen/typescript-operations
```

For Azure Functions:
```bash
pnpm add @azure/functions @azure/cosmos cosmos-orm ulidx
```

## Git Setup

```bash
git init
git add .
git commit -m "init"
```

Conventional commits are not required, but commit messages should be descriptive.
