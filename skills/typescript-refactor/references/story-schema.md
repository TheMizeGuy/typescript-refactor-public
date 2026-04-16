# Scrum-master Story Schema (shared reference)

This file is the single source of truth for the per-story markdown schema emitted by both the main `typescript-refactor` skill and the `stories` handoff skill.

## Story file path

`<board-path>/<id>.md` -- one story per file. Board path resolution:

1. Inline `--board=<path>` argument (stories skill only)
2. `.claude/scrum-master.local.md` -> `board_path:` field
3. `.claude/typescript-refactor.local.md` -> `scrum_master_board_path:` field
4. Auto-detect: glob `**/Backlog/current/` then `**/Backlog/`, take the first hit's parent
5. Default: `<PROJECT_ROOT>/Backlog/current/`

## ID format

`<STORY_PREFIX>-<EPIC>-<SEQ>` -- default prefix `TSR`, epic from the table below, SEQ zero-padded to 2 digits.

If a board already has a story at the intended ID, append `A` suffix (`TSR-F-01A`) and rewrite all dependency references consistently.

## Epic prefixes

| Epic | Prefix | Meaning |
|---|---|---|
| F | Foundation | tsconfig, lint, CI, baseline gates |
| B | Boundaries | Zod placement at runtime trust boundaries |
| R | Renames | JS -> TS file renames with bucket-driven effort |
| FW | Framework | Framework migration (Express -> Fastify, Jest -> Vitest, etc.) |
| C | Cleanup | Final strictness, remove `allowJs`, debt closure |

## Tag rules

| Slice epic | Tags |
|---|---|
| F | `typescript-refactor`, `foundation`, `tsconfig` (or `lint`, `ci` as appropriate) |
| B | `typescript-refactor`, `boundaries`, `zod` (or `env-schema`, `auth-claims`) |
| R | `typescript-refactor`, `rename`, `bucket-<n>` |
| FW | `typescript-refactor`, `framework-migration`, `<from>-to-<to>` |
| C | `typescript-refactor`, `cleanup`, `debt-closure` |

## Frontmatter schema (canonical)

```yaml
---
id: <STORY_PREFIX>-<EPIC>-<SEQ>
title: <concise title>
state: Backlog
owner: ""
priority: <P0|P1|P2|P3>
effort: <S|M|L|XL>
risk: <low|medium|high>
epic: <F|B|R|FW|C>
class: standard
scope:
  include:
    - "<glob -- workspace-relative>"
  exclude:
    - "<glob>"
acceptance:
  - "<mechanically verifiable assertion with action verb>"
dependencies:
  blocks: [<slice ids that this slice unblocks>]
  blocked-by: [<slice ids that must complete first>]
evidence:
  commit: ""
  pr: ""
  test_output: ""
  screenshot: ""
created: <YYYY-MM-DD>
updated: <YYYY-MM-DD>
tags: [typescript-refactor, <epic>, <extra-tags>]
---
```

## Body schema (canonical)

```markdown
# <id>: <title>

## Description
<2-5 sentences from the slice description: what and why>

## Technical notes
<file paths, patterns to follow, constraints>

## Acceptance criteria
<expanded from frontmatter; use Given-When-Then if the slice has complex preconditions>

## Out of scope
<explicit exclusions>

## Cutover strategy
<one of: no-risk | shadow-compare | parity-harness | route-cohort | big-bang | strangler>

## Rollback
<specific mechanism -- must be mechanical, never "fix forward" or "heroic">

## Verification commands
<exact commands to run for AC verification>
```

## AC quality rule

Every acceptance criterion MUST contain at least one action verb from this set:

```
exists, returns, passes, reports, contains, matches, succeeds, fails, exit 0, exit code,
>=, <=, >0 hit, 0 hits, tsc --noEmit, type-coverage, eslint, vitest, grep
```

If an AC is prose-only ("works correctly", "is faster"), the slicer must rephrase or the verifier must block.

## Size guard (rejection rule)

A slice MUST be split before becoming a story if any of:

- AC count > 5
- `scope.include` glob count > 10 (after expansion)
- Implementation likely to span more than one logical commit

## Worked example: TSR-F-01

```markdown
---
id: TSR-F-01
title: Install TypeScript and write baseline tsconfig
state: Backlog
owner: ""
priority: P0
effort: S
risk: low
epic: F
class: standard
scope:
  include:
    - "package.json"
    - "tsconfig.json"
  exclude:
    - "src/**"
acceptance:
  - "package.json devDependencies contains typescript >=5.4"
  - "tsconfig.json exists at repo root"
  - "tsc --noEmit exit 0 against existing JS with allowJs=true"
  - "npx type-coverage --at-least 50 succeeds (baseline measurement)"
dependencies:
  blocks: [TSR-F-02, TSR-F-03, TSR-F-04, TSR-F-05]
  blocked-by: []
evidence:
  commit: ""
  pr: ""
  test_output: ""
  screenshot: ""
created: 2026-04-14
updated: 2026-04-14
tags: [typescript-refactor, foundation, tsconfig]
---

# TSR-F-01: Install TypeScript and write baseline tsconfig

## Description
Add TypeScript to devDependencies and write the baseline tsconfig with `allowJs: true, strict: false, noEmit: true, skipLibCheck: true`. This is the no-behavioral-change foundation slice -- every other slice depends on it.

## Technical notes
Use the package manager already in the repo (detected by lockfile). Pin TypeScript to the latest stable 5.x. Do NOT enable strict mode in this slice.

## Acceptance criteria
- Given the repo has no tsconfig at root, when this slice ships, then `tsconfig.json` exists with the agreed baseline shape.
- `pnpm tsc --noEmit` (or equivalent) returns exit 0.
- `npx type-coverage` reports a non-zero baseline percentage.
- No file under `src/` was modified in this slice.

## Out of scope
- Renaming any `.js` files
- Enabling any strict flag
- Adding lint or CI configuration

## Cutover strategy
no-risk

## Rollback
Revert the PR. No runtime behavior change.

## Verification commands
- `pnpm tsc --noEmit` -- must exit 0
- `npx type-coverage --detail --strict --ignore-catch | tail -5` -- capture baseline %
- `git diff src/ | wc -l` -- must equal 0
```
