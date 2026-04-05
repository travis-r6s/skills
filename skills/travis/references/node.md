---
name: node
description: Travis's Node.js patterns — built-in modules, file system, environment, process, CLI scripts, Azure Functions. Use when writing Node.js scripts, servers, or CLI tools.
---

# Node.js Patterns

## Module Imports

Always use the `node:` protocol for built-ins:

```ts
import { readFile, writeFile, mkdir } from 'node:fs/promises'
import { existsSync, createReadStream } from 'node:fs'
import { join, dirname, resolve, extname } from 'node:path'
import { fileURLToPath } from 'node:url'
import { setTimeout } from 'node:timers/promises'
```

For `__dirname` equivalent in ESM:
```ts
import { fileURLToPath } from 'node:url'
import { dirname, join } from 'node:path'

const __filename = fileURLToPath(import.meta.url)
const __dirname = dirname(__filename)
```

## File System

```ts
import { readFile, writeFile, mkdir, readdir } from 'node:fs/promises'
import { existsSync } from 'node:fs'
import { join } from 'node:path'

// Read file
const content = await readFile(join(__dirname, 'data.json'), 'utf-8')
const data = JSON.parse(content)

// Write file (ensure directory exists first)
await mkdir(outputDir, { recursive: true })
await writeFile(join(outputDir, 'result.json'), JSON.stringify(result, null, 2))

// Check existence before reading
if (!existsSync(filePath)) throw new Error(`File not found: ${filePath}`)

// Read directory
const files = await readdir(dir)
const jsonFiles = files.filter(f => f.endsWith('.json'))
```

## Environment Variables

Always via Doppler — never `.env` files. Access via `process.env`:

```ts
const dsn = process.env.SENTRY_DSN
// For required vars, guard immediately:
const apiKey = process.env.SHOPIFY_API_KEY
if (!apiKey) throw new Error('SHOPIFY_API_KEY is required')
```

Use `defu` to build config objects with defaults:
```ts
import { defu } from 'defu'

const config = defu(
  { apiKey: process.env.API_KEY, timeout: Number(process.env.TIMEOUT) },
  { timeout: 30_000 },
)
```

## HTTP: Native Fetch

Always native `fetch` — never `axios`:

```ts
const response = await fetch('https://api.example.com/orders', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json', Authorization: `Bearer ${token}` },
  body: JSON.stringify({ query }),
})

if (!response.ok) throw new Error(`Request failed: ${response.status} ${response.statusText}`)

const data = await response.json() as OrdersResponse
```

## Process & Exit

```ts
// Graceful exit with message
logger.success('Done!')
process.exit(0)

// Handle user cancellation
const value = await consola.prompt('Continue?', { type: 'confirm' })
if (!value) {
  logger.warn('Cancelled.')
  process.exit(0)
}

// Unhandled rejections (top-level in long-running scripts)
process.on('unhandledRejection', (error) => {
  Sentry.captureException(error)
  process.exit(1)
})
```

## Script Entry Pattern

Interactive scripts with prompts at the top level:

```ts
// src/index.ts
import consola from 'consola'
import { ulid } from 'ulidx'
import * as Orchestrator from './1.step/orchestrator'

const logger = consola.withTag('My Script')

logger.box('My Script v1.0')

const startDate = await consola.prompt('Start date (YYYY-MM-DD):', { type: 'text', required: true })
if (!startDate) { logger.warn('Cancelled.'); process.exit(0) }

logger.start('Running...')
const result = await Orchestrator.orchestrator({ id: ulid(), startDate })
logger.success(`Done: ${result.id}`)
```

Run with: `doppler run -- tsx src/index.ts`

## CLI Tools: citty

For tools that need subcommands:

```ts
import { defineCommand, runMain } from 'citty'

const main = defineCommand({
  meta: { name: 'my-tool', version: '1.0.0', description: 'Does things' },
  subCommands: {
    export: () => import('./commands/export').then(m => m.default),
    import: () => import('./commands/import').then(m => m.default),
  },
})

runMain(main)
```

Individual command:
```ts
// commands/export.ts
import { defineCommand } from 'citty'

export default defineCommand({
  meta: { description: 'Export data' },
  args: {
    from: { type: 'string', description: 'Start date', required: true },
    to: { type: 'string', description: 'End date', required: true },
  },
  run: async ({ args }) => {
    const logger = consola.withTag('export')
    logger.start(`Exporting from ${args.from} to ${args.to}`)
    // ...
  },
})
```

## Timers

```ts
import { setTimeout as sleep } from 'node:timers/promises'

await sleep(1000)  // non-blocking sleep

// With AbortController
const ac = new AbortController()
await sleep(5000, undefined, { signal: ac.signal })
```

## Azure Functions

Register functions via the v4 programming model:

```ts
// src/functions/my-function/index.ts
import { app } from '@azure/functions'
import { handler } from './handler'

app.http('my-function', {
  methods: ['POST'],
  authLevel: 'function',
  route: 'my-route',
  handler,
})
```

```ts
// src/functions/my-function/handler.ts
import type { HttpRequest, HttpResponseInit, InvocationContext } from '@azure/functions'

export async function handler(req: HttpRequest, context: InvocationContext): Promise<HttpResponseInit> {
  const body = await req.json() as MyPayload
  // process...
  return { status: 200, jsonBody: result }
}
```

See [architecture.md](architecture.md) for full Azure Functions project structure.

<!--
Source references:
- https://nodejs.org/api/esm.html
- https://nodejs.org/api/fs.html
- https://azure.github.io/azure-functions-nodejs-worker/
-->
