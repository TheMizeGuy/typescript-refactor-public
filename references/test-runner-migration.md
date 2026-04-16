# Test Runner Migration Reference

Practitioner reference for testing TypeScript code. Covers runner selection, type-level testing, mocking, mutation testing, and E2E.

## Runner Comparison

| Runner | Native TS | Native ESM | Speed | Coverage | Best For |
|---|---|---|---|---|---|
| Vitest 3.x | Yes (esbuild) | First-class | Fast | v8, istanbul | New projects |
| Jest 29+ | With ts-jest/swc | Needs config | Medium | istanbul | Existing Jest codebases |
| Node `--test` | Via strip-types (22+) | Native | Fast | c8 | Zero-dep policies |
| Bun test | Native | Native | Fastest | Built-in | Bun-native apps |
| Deno test | Native | Native | Fast | Built-in | Deno-native apps |
| Mocha 10.x | Via loader | Via loader | Medium | via c8/nyc | Legacy |

## Decision tree

| Situation | Pick |
|---|---|
| Greenfield TypeScript app | Vitest |
| Vite-based app | Vitest (config reuse) |
| Existing Jest suite, no ESM pain | Stay on Jest |
| Existing Jest suite, blocked by ESM | Migrate to Vitest |
| Zero-dependency preference, Node 22+ | `node --test` |
| Bun-native app | Bun test |

## Vitest features that matter

| Feature | What it does |
|---|---|
| `defineConfig({ test })` | Reuses Vite plugin pipeline |
| `--typecheck` | Runs `tsc --noEmit` against `*.test-d.ts` in the same run |
| `test.projects` | Monorepo multi-project (replaces deprecated workspace config) |
| `vitest bench` | Microbenchmark mode |
| `expectTypeOf` | Built-in type assertions |
| Smart watch | Re-runs only affected tests via module graph |

## Jest -> Vitest migration

```bash
npx @vitest/codemod
```

| Jest -> Vitest risk | Mitigation |
|---|---|
| `jest.fn()` / `jest.mock()` | Replace with `vi.fn()` / `vi.mock()` |
| Custom matchers via `expect.extend` | Works in Vitest, re-check |
| Jest setup files | Map to Vitest `setupFiles` |
| `jest.useFakeTimers()` | `vi.useFakeTimers()` |
| ts-jest transforms | Removed -- Vitest uses esbuild |
| Snapshot file locations | Compatible but verify |

## Mutation testing (quality bar)

Coverage % is not quality -- mutation testing is. AI tests routinely show 93% line coverage with 34% mutation score.

| Tool | For |
|---|---|
| `Stryker` | JavaScript/TypeScript |
| `@vitest/coverage-v8` | Coverage basis for mutation budget |

## Type testing

| Tool | How |
|---|---|
| Vitest `--typecheck` | Built-in, `expectTypeOf` |
| `tsd` | Standalone type assertion library |
| `expect-type` | Composable type assertions |

## E2E

| Framework | Use when |
|---|---|
| Playwright | Default for web E2E in 2026 |
| Cypress | Existing suites |
