# Changelog

All notable changes to the `typescript-refactor` plugin will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.1] - 2026-07-05

### Changed

- **Version realignment**: reset this public mirror's version from 1.0.0 to 0.1.1 so it tracks the private source of truth (`TheMizeGuy/typescript-refactor`). Public mirror versions now move in lockstep with the private plugin; the earlier independent public-milestone numbering (1.0.0) is retired. No functional or content change — this bump only aligns the version number.

## [1.0.0] - 2026-07-05

### Added

- Initial public release of the `typescript-refactor` plugin.
- 7 read-only planner agents (`js-inventory-cartographer`, `type-surface-analyst`,
  `runtime-boundary-analyst`, `test-inventory-analyst`, `typescript-target-architect`,
  `migration-slicer`, `plan-verifier`) dispatched in 4 sequential waves by the
  `typescript-refactor` skill, plus `audit`, `target-architecture`, and `stories` skills for
  standalone sub-passes.
- Grounded in 8 embedded TypeScript reference files under `references/` (tsconfig strictness,
  build toolchain, test-runner migration, monorepo patterns, migration patterns, runtime
  boundary patterns, modern TS features, type-surface analysis).
- Scrum-master-format story emission (`skills/typescript-refactor/references/story-schema.md`)
  with dependency DAG, AC quality validation, and a size guard (max 5 AC / 10 scope files per
  slice).
- `skills/typescript-refactor/references/wave-gates.md` -- the acceptance gate applied after
  every wave: the evidence-label vocabulary (`SOURCE`/`BUILD`/`RUNTIME`/`DOC`/`INFERRED`/
  `TARGET-ASSUMPTION`/`OPEN-QUESTION`) with a worked example distinguishing a valid report row
  from two defective ones, per-wave required-section checks, the gate failure protocol (one
  re-dispatch with defective rows quoted verbatim, then stop), and a failure-mode table.
- "Execution mode" notes on all three orchestrator skills: specialists always inherit the
  session model (always the strongest available Claude, never a fixed pin); for small scopes,
  when the session model is already the strongest available tier, the orchestrator may run a
  wave step inline instead of dispatching, without weakening the read-only / evidence-labeling
  discipline of the specialist it replaces.
- Pre-verifier artifact check and verifier-report validity check so an incomplete or
  unparseable verification pass is never silently treated as a pass.
- `USAGE.md` walkthrough guide and this changelog.
- MIT license.

[1.0.0]: https://github.com/TheMizeGuy/typescript-refactor-public/releases/tag/v1.0.0
