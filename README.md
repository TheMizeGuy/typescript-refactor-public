# typescript-refactor

JavaScript-to-TypeScript refactor planning specialist for [Claude Code](https://docs.anthropic.com/en/docs/build-with-claude/claude-code), running on the session model (always the strongest available Claude). Dispatches a team of 7 read-only senior planner agents in sequential waves to produce an evidence-tagged migration plan, scrum-master-format per-story markdown files, and a Mermaid dependency DAG.

## What this is

A **planning plugin**, not an implementation plugin. It does not write to your codebase. It produces:

1. A consolidated migration plan with evidence labels (`SOURCE` / `BUILD` / `RUNTIME` / `DOC` / `INFERRED` / `TARGET-ASSUMPTION` / `OPEN-QUESTION`) -- see `skills/typescript-refactor/references/wave-gates.md` for the full vocabulary and the acceptance gate applied after each wave
2. A target-state architecture (tsconfig, framework, validation, data layer, test runner, CI gates -- with tsgo, the TypeScript 7 native compiler, as the primary typecheck gate) with reference citations
3. A phased slice catalog (Foundation -> Boundaries -> Renames -> Framework -> Cleanup) with explicit dependencies and rollback plans
4. A Mermaid dependency DAG
5. One scrum-master-format story file per slice

## Installation

```bash
# 1. Add this repo as a marketplace
claude plugin marketplace add https://github.com/TheMizeGuy/typescript-refactor-public.git

# 2. Install the plugin
claude plugin install typescript-refactor@typescript-refactor-public

# 3. Restart Claude Code for the plugin to load
```

After restart, verify with `claude plugin list`. Updates ship through the same channel: when a new release lands, run `claude plugin marketplace update typescript-refactor-public` then `claude plugin update typescript-refactor@typescript-refactor-public`, or accept the update prompt in `/plugin`.

Manual alternative: `git clone https://github.com/TheMizeGuy/typescript-refactor-public.git` and load with `claude --plugin-dir <path>`.

## Skills

| Slash command | What it does |
|---|---|
| `/typescript-refactor [scope]` | Full team planning. Dispatches all 7 agents in 4 waves, produces plan + DAG + stories |
| `/typescript-refactor:audit [scope]` | Wave 1 only. Inventory + type surface report. No plan, no stories |
| `/typescript-refactor:target-architecture` | Target architect only. Decision matrix with reference citations. No slices |
| `/typescript-refactor:stories [plan]` | Stories handoff. Reads existing plan, emits scrum-master-format story files + DAG |

### Scope arguments

| Arg | Meaning |
|---|---|
| empty | Full project (default) |
| `quick` | Reduced team mode (4 agents instead of 7) -- for codebases <1000 JS files |
| `<path>` | Scope to a specific directory or glob |
| `all` | Entire project including rarely-touched dirs |

## Agent fleet

All inherit the session model (always the strongest available Claude), all read-only, dispatched by the main skill orchestrator.

| Agent | Wave | Role |
|---|---|---|
| `js-inventory-cartographer` | 1 | Package/lockfile/build/framework/CI/test/lint inventory |
| `type-surface-analyst` | 1 | Existing type surface depth: .d.ts, @ts-check, JSDoc, escape-hatches, day-one buckets |
| `runtime-boundary-analyst` | 2 | Every runtime trust boundary (HTTP, DB, env, queue, ws, file, auth) for Zod placement |
| `test-inventory-analyst` | 2 | Runner, file count, style quality, parity-harness candidates |
| `typescript-target-architect` | 3 | tsconfig, framework, validation, data layer, lint, monorepo -- full decision matrix |
| `migration-slicer` | 3 | Seam selection, slice catalog, dependency DAG, dispatch waves, cutover + rollback per slice |
| `plan-verifier` | 4 | Claim-to-source audit, DAG acyclicity, AC quality, rollback presence, reference citation coverage |

### Wave dispatch pattern

| Wave | Mode | Agents |
|---|---|---|
| 1 | parallel | inventory + type-surface |
| 2 | parallel | boundaries + tests |
| 3 | sequential | architect -> slicer (slicer reads architect's output) |
| 4 | single | verifier (loops if BLOCKED) |

## Embedded references

The `references/` directory contains distilled TypeScript knowledge that grounds every agent's decisions:

| File | Content |
|---|---|
| `tsconfig-patterns.md` | Strict preset, upgrade order, module resolution, project references |
| `build-toolchain.md` | Tool selection (tsc, swc, esbuild, tsup, Vite, Rspack), speed tiers, framework selection |
| `test-runner-migration.md` | Vitest vs Jest vs Node --test, mutation testing, Jest -> Vitest migration |
| `monorepo-patterns.md` | pnpm workspaces, Turborepo vs Nx, publishing, workspace protocol |
| `migration-patterns.md` | Typed != safe, parse-don't-validate, branded types, migration phases |
| `runtime-boundary-patterns.md` | Boundary taxonomy, Zod placement patterns, validation library selection |
| `modern-ts-features.md` | erasableSyntaxOnly, verbatimModuleSyntax, satisfies, using, decorators |
| `type-surface-analysis.md` | interface vs type, discriminated unions, anti-patterns, type-coverage |

## Settings (optional)

Create `.claude/typescript-refactor.local.md` in your project for per-project preferences:

```markdown
---
story_prefix: "TSR"
scrum_master_board_path: "Backlog/current"
target_framework: "fastify"
target_test_runner: "vitest"
target_data_layer: "drizzle"
target_validation: "zod"
target_monorepo_tool: "turbo"
deployment: "node-lts"
strict_ladder: "all-at-once"
package_manager: "pnpm"
ci_system: "github-actions"
---
```

## Output paths

| Artifact | Location |
|---|---|
| Plan doc | `<PROJECT_ROOT>/docs/typescript-refactor/<YYYY-MM-DD>-<RUN_ID>-plan.md` |
| Mermaid DAG | `<PROJECT_ROOT>/docs/typescript-refactor/<YYYY-MM-DD>-<RUN_ID>-dag.md` |
| Stories | `<board-path>/TSR-*.md` (scrum-master schema) |
| Blackboard (full evidence trail) | `<PROJECT_ROOT>/.claude/typescript-refactor/<RUN_ID>/` |

## Composition with other plugins

| Plugin | When | How |
|---|---|---|
| `scrum-master` | After plan emission | Stories are written in the exact scrum-master schema |
| `dev` | After stories exist | Pulls Ready stories and dispatches agents |
| `typescript-senior-review` | After each migrated slice ships | Review changed files |

## Known limitations

| Limitation | Reason |
|---|---|
| Agents do NOT execute the migration | By design -- execution is separate |
| No timeline promises | Effort is S/M/L/XL only |
| Does not modify your tsconfig | Foundation slice F-01 will when executed |

## License

MIT

## Author

[TheMizeGuy](https://github.com/TheMizeGuy)
