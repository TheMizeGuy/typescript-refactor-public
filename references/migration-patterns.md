# JS-to-TypeScript Migration Patterns

Core migration doctrine: typed != safe, boundary placement, parse-don't-validate, branded types, strictness laddering.

## Part A: Typed != Safe

The TypeScript compiler erases all type information before emit. At runtime you have JavaScript, and JavaScript has no type checking. A cast is an assertion, a declaration file is a promise, and `JSON.parse` returns `any`.

| Claim | Reality |
|---|---|
| `function f(x: User)` -- caller must pass a User | Caller can pass `{} as User` or `JSON.parse(req.body)` |
| `interface Order { total: number }` at a DB boundary | DB row may have `total: "12.50"` (string) or `total: null` |
| `req.body: SignupInput` after body-parse middleware | `req.body` is `any`. Typing it is an assertion, not validation |
| `const env: { DATABASE_URL: string } = process.env` | `process.env` values are `string \| undefined` |

**Rule:** type-check in the editor, validate at runtime. Every boundary crossing must parse, not assume.

## Part B: Parse Don't Validate

Instead of checking that data is valid and passing it along unchanged, transform raw input into a proper domain type whose existence proves validity. Post-parse, the rest of the program doesn't need to re-check.

### Branded types (nominal simulation)

```ts
type Brand<T, B> = T & { readonly __brand: B };
type Email = Brand<string, "Email">;
type UserId = Brand<string, "UserId">;
type OrderId = Brand<string, "OrderId">;
```

Benefits: prevent ID mix-ups at compile time, document intent, no runtime cost, refactor-safe.

### Smart constructors

Pair a branded type with a single constructor function. Consumers must call the constructor.

```ts
export function parseEmail(s: unknown): Email {
  if (typeof s !== "string" || !/.+@.+/.test(s)) throw new TypeError();
  return s as Email;
}
```

### Zod integration

```ts
const EmailSchema = z.string().email().brand<"Email">();
type Email = z.infer<typeof EmailSchema>;
```

## Part C: JS-to-TS Migration Strategy

### Phase 1: Foundation (no behavioral change)

1. Install TypeScript, write baseline tsconfig (`allowJs: true, strict: false, noEmit: true`)
2. Bootstrap typescript-eslint
3. Add CI type-check gate (warn-only at first)
4. Measure type-coverage baseline

### Phase 2: Boundary typing

1. Create typed env schema (`src/env.ts` with Zod)
2. Attach Zod to HTTP ingress routes
3. Type egress responses
4. Type DB boundaries (ORM or Zod at driver)
5. Type auth claims

### Phase 3: Progressive rename

1. Ready-to-rename files (`@ts-check` already passes) -- `git mv .js .ts`
2. Light-polish files -- rename + add annotations
3. JSDoc-first files -- `@ts-check` + JSDoc before rename
4. Blank-slate files -- rename with `any` + follow-up fixes

### Phase 4: Strict ladder

Enable strict flags one at a time in the order from the tsconfig-patterns reference. Use a separate `tsconfig.strict.json` that extends the base and includes progressively more directories.

### Phase 5: Cleanup

1. Turn off `allowJs`
2. Raise type-coverage gate to 95%+
3. Enable `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`
4. Remove remaining `any`
5. Remove all `@ts-nocheck` and justify every `@ts-expect-error`

## Post-migration cleanup grep patterns

```bash
# Find remaining any
grep -rnE ': any\b|as any\b' src/ | grep -v '\.d\.ts'

# Find @ts-ignore (should be @ts-expect-error instead)
grep -rn '@ts-ignore' src/

# Find unsafe JSON.parse without Zod
grep -rnE 'JSON\.parse\(' src/ | grep -v 'Schema\.\(parse\|safeParse\)'

# Find process.env reads not going through env.ts
grep -rnE 'process\.env\.' src/ | grep -v 'src/env'
```

## Evidence labels used throughout migration planning

| Label | Meaning |
|---|---|
| `SOURCE` | Direct file observation |
| `BUILD` | From build metadata |
| `RUNTIME` | Observed at runtime |
| `DOC` | Documented elsewhere |
| `INFERRED` | Reasoned from multiple signals |
| `TARGET-ASSUMPTION` | Chosen target state without direct evidence |
| `OPEN-QUESTION` | Could not resolve; requires user decision |
