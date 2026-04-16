# Type Surface Analysis Patterns

Best practices and idioms for analyzing and improving the type surface during migration.

## interface vs type

| Use | For |
|---|---|
| `interface` | Object shapes, especially when extended (`interface extends` is faster for the checker than `&`) |
| `type` | Unions, intersections, mapped types, template literals, conditional types, primitives |

## Discriminated unions

The most important TypeScript pattern for domain modeling:

```ts
type Result<T, E = Error> =
  | { ok: true; value: T }
  | { ok: false; error: E };
```

Narrowing is automatic via the discriminant field.

## Branded / opaque types

Prevent accidental type confusion between same-shaped values:

```ts
type Brand<T, B> = T & { readonly __brand: B };
type UserId = Brand<string, "UserId">;
type OrderId = Brand<string, "OrderId">;

// deleteOrder(userId) -- compile error
```

## satisfies over annotation

```ts
// Annotation widens:
const routes: Record<string, string> = { home: '/home' };
// routes.home is string

// satisfies preserves:
const routes = { home: '/home' } satisfies Record<string, string>;
// routes.home is "/home"
```

## Exhaustiveness checking

```ts
function assertNever(x: never): never {
  throw new Error(`Unexpected: ${x}`);
}

switch (action.type) {
  case 'increment': return state + 1;
  case 'decrement': return state - 1;
  default: return assertNever(action.type);
}
```

## Common anti-patterns to flag during analysis

| Anti-pattern | Better alternative |
|---|---|
| `x as T` at boundaries | Zod parse |
| `any` in function signatures | `unknown` + narrowing |
| `!` non-null assertion chains | Optional chaining + nullish coalescing |
| `Object.assign({}, x)` for spreading | Typed spread with explicit type |
| `=== undefined` checks on optional | `in` operator or discriminant |
| Return type `void` when result is useful | Explicit return type |
| `enum` | `as const` object or union type |
| Namespace | ES module |
| Constructor parameter properties | Explicit fields (if `erasableSyntaxOnly`) |

## Type coverage measurement

```bash
# Install
pnpm add -D type-coverage

# Measure baseline
npx type-coverage --detail --strict --ignore-catch

# CI gate (ratchet up over time)
npx type-coverage --at-least 85
```

Type coverage measures the percentage of identifiers that have a non-`any` type. Use it as a CI gate that ratchets up as migration progresses.

## JSDoc patterns that TypeScript accepts

TypeScript's `checkJs` mode accepts JSDoc annotations:

```js
// @ts-check

/** @type {import('./types').User} */
const user = getUser();

/**
 * @param {string} name
 * @param {number} age
 * @returns {import('./types').User}
 */
function createUser(name, age) { ... }

/** @typedef {{ id: string; email: string }} User */
```

Files with `@ts-check` + full JSDoc are "ready-to-rename" candidates.
