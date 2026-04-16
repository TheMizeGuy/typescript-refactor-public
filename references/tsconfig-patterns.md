# tsconfig Patterns Reference

Distilled from TypeScript 5.x-6.0 compiler documentation. Canonical lookup for tsconfig flags, strict-family upgrade order, module resolution, and project references.

## Recommended strict target

```jsonc
{
  "compilerOptions": {
    // Strictness -- all ON in target state
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitOverride": true,
    "noFallthroughCasesInSwitch": true,
    "noPropertyAccessFromIndexSignature": true,

    // Hygiene
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "forceConsistentCasingInFileNames": true,

    // Module system
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "verbatimModuleSyntax": true,
    "isolatedModules": true,
    "resolveJsonModule": true,

    // Target
    "target": "ES2022",
    "lib": ["ES2022"],
    "erasableSyntaxOnly": true,

    // Emit
    "outDir": "./dist",
    "rootDir": "./src",
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,

    // JS interop (migration phase only)
    "allowJs": true,
    "skipLibCheck": true
  }
}
```

## Strict-family upgrade order (for legacy migration)

| Step | Flag | Why this order |
|---|---|---|
| 1 | `noImplicitAny` | Forces you to confront all untyped surface area first |
| 2 | `strictNullChecks` | Biggest semantic win; requires fixing null/undefined handling everywhere |
| 3 | `strictFunctionTypes` | Usually cheap after the above |
| 4 | `strictBindCallApply` | Almost always zero errors |
| 5 | `strictPropertyInitialization` | Depends on class-heavy code |
| 6 | `noImplicitThis`, `alwaysStrict` | Almost always zero errors |
| 7 | `useUnknownInCatchVariables` | Low cost; narrow `err` with `instanceof Error` |
| 8 | `strict: true` | Promote the explicit flags to the single master switch |
| 9 | `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`, `noImplicitOverride` | Add one at a time |

## Strictness flags reference

| Flag | In `strict`? | What it catches |
|---|---|---|
| `noImplicitAny` | Yes | Untyped variables/parameters |
| `strictNullChecks` | Yes | Implicit null/undefined assignment |
| `strictFunctionTypes` | Yes | Unsound callbacks |
| `strictBindCallApply` | Yes | bind/call/apply argument mismatch |
| `strictPropertyInitialization` | Yes | Uninitialized class fields |
| `noImplicitThis` | Yes | Untyped `this` |
| `alwaysStrict` | Yes | JS strict mode |
| `useUnknownInCatchVariables` | Yes | catch(err) as unknown |
| `exactOptionalPropertyTypes` | No | `a?: number` vs `a: number\|undefined` |
| `noUncheckedIndexedAccess` | No | `obj[k]` returns `T\|undefined` |
| `noImplicitOverride` | No | Subclass override keyword |

## Module + moduleResolution combinations

| Scenario | `module` | `moduleResolution` |
|---|---|---|
| Node.js app (tsc compiles) | `nodenext` | `nodenext` |
| Library for Node (tsc-emitted) | `nodenext` | `nodenext` |
| Bundler app (Vite, webpack, Next) | `preserve` or `esnext` | `bundler` |
| Bun app | `preserve` | `bundler` |
| tsup-bundled library | `esnext` | `bundler` |
| Node type-stripping | `nodenext` + `verbatimModuleSyntax` + `erasableSyntaxOnly` | `nodenext` |

## Project references (monorepo pattern)

```jsonc
// packages/core/tsconfig.json
{
  "compilerOptions": { "composite": true, "declaration": true, "outDir": "./dist" }
}

// root tsconfig.json
{
  "files": [],
  "references": [
    { "path": "./packages/core" },
    { "path": "./apps/web" }
  ]
}
```

`tsc -b` walks the reference graph, compiles leaves first, and uses `.tsbuildinfo` for incremental.

## Incremental builds

| Feature | Detail |
|---|---|
| `--incremental` | Writes `.tsbuildinfo` recording file hashes. Next run reuses where signatures haven't changed |
| `composite` | Always produces `.tsbuildinfo`. Required by `tsc --build` |
| Fine-grained gate | `assumeChangesOnlyAffectDirectDependencies: true` restricts invalidation to direct importers |
