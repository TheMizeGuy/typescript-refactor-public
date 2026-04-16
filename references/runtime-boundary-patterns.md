# Runtime Boundary Patterns

Where to place Zod parsing and branded types during a JS-to-TS migration. Covers boundary taxonomy, validation library selection, and placement patterns.

## Boundary taxonomy

| Class | Examples | Target-state pattern |
|---|---|---|
| `HTTP_INGRESS` | Express `req.body`, Fastify handler input | Zod schema at handler entry |
| `HTTP_EGRESS_RESPONSE` | `fetch` / `undici` / `axios` response bodies | Zod parse on `.json()` result |
| `CLI_ARGS` | `process.argv`, `commander`, `yargs` | Typed parser with Zod fallthrough |
| `ENV_VAR` | `process.env.FOO` reads | Single-file typed env schema |
| `DB_RESULT` | `pg.query`, `mongodb` collection reads | ORM OR Zod at driver boundary |
| `QUEUE_MSG` | BullMQ `job.data`, Kafka payloads | Zod parse at job handler entry |
| `WEBSOCKET_IN` | `ws` message events, Socket.IO | Zod parse per-event |
| `FILE_READ` | `fs.readFile` returning JSON/YAML | Zod parse after deserialization |
| `EXTERNAL_CALLBACK` | Third-party SDK callbacks | Zod parse on callback arg |
| `USER_PROVIDED_CODE` | `eval`, `new Function`, plugin loader | Cannot parse -- isolate |
| `INTERNAL_IPC` | `worker_threads` messages | Zod or branded type |
| `STORAGE_REHYDRATION` | `localStorage`, IndexedDB, cookies | Zod parse on deserialized value |
| `AUTH_CLAIMS` | JWT verify result, session lookups | Zod parse on decoded payload |

## Validation library selection

| Need | Pick | Bundle size | Speed |
|---|---|---|---|
| Default (best DX, largest community) | `zod` v4 | ~14KB min | Good |
| Edge runtime, smallest bundle | `valibot` | ~1-5KB (tree-shakeable) | Good |
| Fastest runtime, JIT-compiled | `arktype` | ~24KB | Fastest |
| JSON Schema output + TS types | `typebox` | ~10KB | Fast |
| Compile-time (transformer) | `typia` | 0KB runtime | N/A (compile-time) |
| Effect ecosystem | `@effect/schema` | Part of Effect | Good |

## Placement patterns

### HTTP ingress (Express example)

```ts
import { z } from 'zod';

const CreateUserSchema = z.object({
  email: z.string().email(),
  name: z.string().min(1),
});

app.post('/users', (req, res) => {
  const body = CreateUserSchema.parse(req.body);
  // body is now typed and validated
});
```

### Typed env schema

```ts
import { z } from 'zod';

const EnvSchema = z.object({
  DATABASE_URL: z.string().url(),
  PORT: z.coerce.number().default(3000),
  NODE_ENV: z.enum(['development', 'production', 'test']),
});

export const env = EnvSchema.parse(process.env);
```

### HTTP egress

```ts
const UserResponseSchema = z.object({
  id: z.string(),
  email: z.string(),
});

const data = UserResponseSchema.parse(await res.json());
```

### DB boundary with raw driver

```ts
const UserRowSchema = z.object({
  id: z.string(),
  email: z.string(),
  created_at: z.coerce.date(),
});

const result = await pool.query('SELECT * FROM users WHERE id = $1', [id]);
const user = UserRowSchema.parse(result.rows[0]);
```

## Detection heuristics for boundary sweeps

### HTTP ingress
```bash
# Express
grep -rnE "(app|router)\.(get|post|put|patch|delete)\(" src/
# Fastify
grep -rnE "fastify\.(get|post|put|patch|delete|route)\(" src/
# Next.js App Router
grep -rlnE "^export (async )?function (GET|POST|PUT|PATCH|DELETE)\(" src/app/
```

### DB boundaries
```bash
grep -rnE "\.(query|execute)\(['\"\`]" src/
grep -rnE "\.(find|findOne|aggregate)\(" src/
```

### Env vars
```bash
grep -rnE "process\.env\.[A-Z_]+" src/ | sort | uniq -c | sort -rn
```

### Hostile zones (cannot type)
```bash
grep -rnE "\b(eval|new Function)\b" src/
grep -rnE "require\(\s*[\$\w]" src/   # dynamic require
```
