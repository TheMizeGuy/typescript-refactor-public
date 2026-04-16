# Monorepo Patterns Reference

Architecture reference for TypeScript monorepos: workspace tooling, task orchestration, project references, and publishing.

## Package Manager Comparison

| Manager | Workspaces | Speed | Strict isolation | When to pick |
|---|---|---|---|---|
| **pnpm** | First-class | Very fast | Yes (default) | Default for monorepos in 2026 |
| **npm** | Built-in since v7 | Slower (v11 is 65% faster) | No | Zero-migration baseline |
| **bun** | npm-compatible | Fastest | Partial | Speed dominates, bun runtime |
| **yarn berry** | PnP | Fast but ecosystem friction | Yes | Special isolation needs |

## pnpm workspace setup

Root `package.json`:
```json
{
  "private": true,
  "packageManager": "pnpm@10.8.0",
  "scripts": {
    "build": "turbo run build",
    "dev": "turbo run dev"
  }
}
```

`pnpm-workspace.yaml`:
```yaml
packages:
  - "apps/*"
  - "packages/*"
  - "!**/dist/**"
```

## Task orchestration: Turborepo vs Nx

| | Turborepo | Nx |
|---|---|---|
| Config | `turbo.json` | `nx.json` |
| Caching | File-hash, remote | Same + more granular |
| Complexity | Low | Higher |
| Best for | Most monorepos | Complex governance, 50+ packages |

## Workspace protocol

| Specifier | Dev resolution | Published as |
|---|---|---|
| `"workspace:*"` | Local workspace | Exact version at publish |
| `"workspace:^"` | Local workspace | Caret range |
| `"workspace:~"` | Local workspace | Tilde range |

## Project references for TypeScript

Every workspace package gets its own `tsconfig.json` with `composite: true`. Root tsconfig references all packages. `tsc --build` walks the graph, compiles leaves first.

Key flags:
- `composite: true` -- required for references
- `declaration: true` -- required by composite
- `declarationMap: true` -- enables go-to-definition across packages

## Publishing

| Tool | Purpose |
|---|---|
| `changesets` | Version automation, changelogs |
| `attw` (arethetypeswrong) | Types publishing linter |
| `publint` | Package.json publishing linter |
| `api-extractor` | Declaration bundling for single-file d.ts |

## Dual ESM/CJS publishing

Use `tsup --format esm,cjs --dts` or `tsdown`. Package.json:

```json
{
  "type": "module",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.js",
      "require": "./dist/index.cjs"
    }
  }
}
```

## When NOT to monorepo

- Truly independent release cadences
- Truly isolated teams/compliance (PCI scope)
- Different primary languages (Go + TS)
- Open-source libraries consumed externally
