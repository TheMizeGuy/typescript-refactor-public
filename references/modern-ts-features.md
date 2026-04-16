# Modern TypeScript Features Reference

Key features from TypeScript 5.0 through 6.0 that affect migration planning.

## erasableSyntaxOnly (5.8+)

Blocks TypeScript syntax that cannot be erased to produce valid JavaScript: `enum`, `namespace`, constructor parameter properties, `import X = require("...")`. Required for Node's `--experimental-strip-types`.

| Blocked syntax | Alternative |
|---|---|
| `enum Direction { Up, Down }` | `const Direction = { Up: 0, Down: 1 } as const` |
| `namespace Foo { }` | ES module |
| `constructor(private x: number)` | Explicit field + assignment |
| `import X = require("...")` | `import X from "..."` or dynamic `import()` |

Enable: `"erasableSyntaxOnly": true` in tsconfig. Use when targeting Node type-stripping or when you want strict JS interop.

## verbatimModuleSyntax (5.0+)

Rewrites nothing: `import type` stays `import type`, value imports stay. Removes import elision. Supersedes `importsNotUsedAsValues` and `preserveValueImports`.

Forces explicit `import type` hygiene. Required for most modern bundlers and Node type-stripping.

## const type parameters (5.0+)

```ts
function routes<const T extends readonly string[]>(paths: T): T { return paths; }
const r = routes(['/a', '/b']); // readonly ["/a", "/b"] -- not string[]
```

Use when literal inference matters (config objects, route definitions).

## satisfies operator (4.9+)

Verify a value matches a type WITHOUT widening. Keeps the narrow inferred type.

```ts
const config = {
  port: 3000,
  host: 'localhost',
} satisfies Record<string, string | number>;
// typeof config.port is 3000, not string | number
```

Prefer `satisfies` over annotation for literal-inferred values.

## using / await using (5.2+)

Explicit resource management (TC39 stage 3). Automatically calls `[Symbol.dispose]()` or `[Symbol.asyncDispose]()` at scope exit.

```ts
function readFile() {
  using file = openFile('data.txt');
  // file is automatically closed at scope exit
  return file.read();
}
```

Use when managing resources (file handles, DB connections, locks).

## NoInfer<T> (5.4+)

Prevents a type parameter from being inferred at a specific position. Useful for library authors.

```ts
function createSignal<T>(value: T, defaultValue: NoInfer<T>): T { ... }
```

## Inferred type predicates (5.5+)

TypeScript can now automatically infer type predicates for simple filter/guard functions:

```ts
const nums = [1, null, 2, undefined, 3].filter(x => x != null);
// TypeScript 5.5+: nums is number[] (not (number | null | undefined)[])
```

Do not hand-write type predicates when the compiler infers them.

## Stage-3 decorators (5.0+)

Native ECMAScript decorators (not the legacy experimental ones). Use when replacing NestJS experimental decorators or adding cross-cutting concerns.

```ts
function logged(originalMethod: Function, context: ClassMethodDecoratorContext) {
  return function (this: unknown, ...args: unknown[]) {
    console.log(`Calling ${String(context.name)}`);
    return originalMethod.apply(this, args);
  };
}
```

NestJS uses experimental decorators and is NOT compatible with `erasableSyntaxOnly`.

## TypeScript 6.0 migration surface

Key defaults changed in TS 6.0:
- `isolatedModules: true` by default
- `moduleDetection: "auto"` by default
- Several stricter behaviors enabled by default

See the TypeScript blog for the full list of default changes between 5.x and 6.0.

## interface vs type

| Prefer | When |
|---|---|
| `interface` | Shared shapes, object types (better checker performance with `extends`) |
| `type` | Unions, intersections, mapped types, template literals, primitives |

Rule: `interface` for objects you extend, `type` for everything else.
