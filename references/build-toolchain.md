# Build Toolchain Reference

Distilled guide to TypeScript build tools: tsc, swc, esbuild, tsup, tsdown, Vite, Rspack, Bun, tsx, and more.

## Tool Selection Decision Tree

| Scenario | Primary tool | Support tool |
|---|---|---|
| Publish a library (ESM + CJS + d.ts) | `tsup` or `tsdown` | `tsc --noEmit` for type-check |
| Zero-dep, ESM-only library | `tsc` emit-only | none |
| Build a Node server / CLI | `tsup` or `tsx` dev + `tsc --noEmit` | `tsx watch` or `node --watch` |
| Build a React/Vue/Svelte SPA | `Vite` | `tsc --noEmit` in parallel |
| Legacy webpack project | `webpack` + `swc-loader` | `fork-ts-checker-webpack-plugin` |
| Rust-powered webpack replacement | `Rspack` or `Rsbuild` | `builtin:swc-loader` |
| Next.js, Remix, TanStack Start | framework-native | nothing extra |
| Monorepo with many packages | `turborepo` or `nx` | `.tsbuildinfo` cache per package |
| Standalone binary from TS | `Bun build --compile` | none |
| Run a one-off script | `tsx`, `bun`, `deno`, or `node --experimental-strip-types` | none |
| Tight type-check loop in CI | `tsc --build --incremental` | `.tsbuildinfo` |
| Edge / Cloudflare Worker | `Wrangler` (esbuild) or `Vite` | `tsc --noEmit` |

**Baseline heuristic:** for any library, pick `tsup` or `tsdown`. For any app, pick `Vite`. For any script or server in dev, pick `tsx` or `bun`.

## Tools Comparison

| Tool | Lang | Type-check | Transform | Bundle | d.ts emit | Primary use |
|---|---|---|---|---|---|---|
| `tsc` | TS | yes | yes | no | yes | Canonical type-checking, d.ts |
| `swc` | Rust | no | yes | no | WIP | Fast transforms for loaders |
| `esbuild` | Go | no | yes | yes | no | Single-file bundling |
| `tsup` | JS (esbuild) | `--dts` uses tsc | via esbuild | yes | yes | Library bundler |
| `tsdown` | JS (Rolldown) | `--dts` via Oxc | via Rolldown | yes | yes | Rolldown-based library bundler |
| `Vite` | JS | no (parallel) | via esbuild | yes (Rollup) | via plugin | App dev server + build |
| `Rspack` | Rust | no | via swc | yes | no | Rust webpack drop-in |
| `Bun.build` | Zig | no | yes | yes | yes | Bun-native bundling |
| `tsx` | JS (esbuild) | no | yes | no | no | Run TS directly |

## Speed tiers (rough, 10k-line TS codebase, cold)

| Tier | Tools | Time |
|---|---|---|
| Zig native | Bun | ~50-150ms |
| Go native | esbuild | ~200-500ms |
| Rust native | swc, Rspack, Rolldown | ~300-800ms |
| JS on Rust/Go | Vite dev, tsup, tsdown | ~500ms-2s |
| JS on tsc | ts-loader, tsc itself | ~15-60s |

## Fast CI pattern: emit-only + separate check

```json
{
  "scripts": {
    "build": "tsc -p tsconfig.build.json",
    "typecheck": "tsc --noEmit",
    "ci": "npm-run-all --parallel typecheck build"
  }
}
```

## HTTP framework selection

| Existing framework | Default target | Why |
|---|---|---|
| Express 4 | Fastify 5 (throughput + explicitness) OR Hono (if edge) OR Express 5 (lowest cost) |
| Koa | Hono | Similar composition model |
| Hapi | Fastify | Closer mental model |
| NestJS | Keep NestJS (DI investment) -- NOT compatible with `erasableSyntaxOnly` |
