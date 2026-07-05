# typescript-refactor -- Usage Guide

A walkthrough of each skill: what to type, what happens, and what you get back. For the file
map and agent reference, see [`README.md`](README.md).

## Quickstart

1. **Install** -- clone the repo and point Claude Code at it (see [Installation](README.md#installation) in the README).
2. **First invocation** -- from a JavaScript project, run:
   ```
   /typescript-refactor
   ```
3. **What to expect** -- the skill resolves your project's scope, creates a run workspace under
   `.claude/typescript-refactor/<RUN_ID>/` (the "blackboard" -- every intermediate report gets
   written there so a compaction or session reset never loses work), then dispatches the 7
   specialist agents in 4 sequential waves. A full-project run on a mid-size codebase (a few
   hundred JS files) typically takes several minutes; each wave's reports are persisted as they
   complete. At the end you get a migration plan, a Mermaid dependency DAG, and one
   scrum-master-format story file per migration slice.

## Walkthrough: `/typescript-refactor` -- full plan

**You type:**
```
/typescript-refactor
```

**What happens:**

1. The skill gathers project facts (JS/TS file counts, package manager, framework, CI system) and creates the blackboard.
2. Wave 1 dispatches `js-inventory-cartographer` and `type-surface-analyst` in parallel. Their reports are checked against the Wave 1 gate in `skills/typescript-refactor/references/wave-gates.md` -- every fact row must carry an evidence label (`SOURCE`, `BUILD`, ...) naming a re-checkable file or command output.
3. Wave 2 dispatches `runtime-boundary-analyst` and `test-inventory-analyst` in parallel, gated the same way.
4. Wave 3 dispatches `typescript-target-architect`, then (sequentially, since it consumes the architect's output) `migration-slicer`.
5. The skill synthesizes a draft plan, then dispatches `plan-verifier`. A `BLOCKED` verdict triggers a re-dispatch of the implicated specialist (max 2 iterations).
6. On `PASS` or `PASS_WITH_OPEN_QUESTIONS`, the skill writes the final plan doc and DAG under `docs/typescript-refactor/`, then emits one story file per migration slice.

**You get:**

- `docs/typescript-refactor/<date>-<run-id>-plan.md` -- the phased migration plan with evidence-tagged decisions
- `docs/typescript-refactor/<date>-<run-id>-dag.md` -- the Mermaid dependency graph
- `<board-path>/TSR-*.md` -- one scrum-master-format story per slice, ready for `/scrum-master status` or `/dev`
- Open questions the specialists could not resolve, called out explicitly rather than silently guessed

## Walkthrough: `/typescript-refactor:audit` -- inventory only

**You type:**
```
/typescript-refactor:audit
```

**What happens:** only Wave 1 runs (`js-inventory-cartographer` + `type-surface-analyst`, in parallel), gated the same way as the full run, then synthesized into a standalone audit report. No target architecture, no migration slices, no stories.

**You get:** `docs/typescript-refactor/<date>-<run-id>-audit.md` plus a compact summary (JS/TS file counts, type coverage estimate, day-one migration bucket sizes). Use this when you want a baseline before committing to a full planning session, or to answer "how big is this migration, roughly?"

## Walkthrough: `/typescript-refactor:target-architecture` -- target state only

**You type:**
```
/typescript-refactor:target-architecture --constraints="prefer fastify, keep pnpm"
```

**What happens:** the skill optionally reads any existing audit on the blackboard, then dispatches `typescript-target-architect` alone. The agent reads the embedded reference files under `references/`, cites `context7` for any third-party library it recommends, and returns a full decision matrix (TS version, tsconfig, build toolchain, lint, validation library, data layer, HTTP framework, test runner, monorepo tooling, CI gates) -- every row evidence-tagged and reference-cited.

**You get:** `docs/typescript-refactor/<date>-<run-id>-target-architecture.md` with headline choices printed in the summary. Use this to compare target-stack options before locking in a full plan, or to pin down the target for a migration that is already partway done.

## Walkthrough: `/typescript-refactor:stories` -- stories from an existing plan

**You type:**
```
/typescript-refactor:stories latest
```

**What happens:** the skill locates the most recent `plan-final.md` on the blackboard (or a path you give it), resolves the scrum-master board path, validates the slice catalog (rejecting any slice with a prose-only acceptance criterion or a missing rollback), scans the board for ID collisions, then writes one story file per slice using the canonical schema in `skills/typescript-refactor/references/story-schema.md`.

**You get:** `<board-path>/TSR-*.md` story files plus `<board-path>/../board-dag.md`, the color-coded Mermaid dependency graph. Use this to re-emit stories after editing a plan by hand, or to hand an existing plan to a project that did not run the full `/typescript-refactor` skill.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Skill says "this codebase is already TypeScript" and stops | Zero `.js`/`.jsx`/`.mjs`/`.cjs` files matched in scope | Pass an explicit `<path>` argument if the JS lives in a subdirectory the default scope misses |
| A wave's report reads as rich prose with no evidence labels | The specialist ignored its report template | The orchestrator re-dispatches once, quoting the template section; if it fails again the run stops and reports the exact defect -- rerun with a narrower scope |
| Verifier returns `BLOCKED` citing missing artifacts | A wave report was never persisted to the blackboard before the verifier ran | Check `.claude/typescript-refactor/<RUN_ID>/` for the missing file; rerun the skill (each wave's reports persist independently, so most of the run does not need to repeat) |
| Stories fail to write -- "missing rollback" or "prose-only AC" | The migration-slicer produced a slice that fails the story-schema validation gate | Re-run `/typescript-refactor:target-architecture` and `/typescript-refactor` for that scope, or hand-edit `plan-final.md`'s slice catalog before re-running `/typescript-refactor:stories` |
| Two agents report different counts for the same thing (routes, files, tests) | Different scope interpretation between specialists | Expected and handled: both counts are recorded in the draft plan and the verifier's consistency audit adjudicates -- check the plan's Open Questions section |
| `quick` mode skipped a wave I wanted | `quick` dispatches only inventory + target-architecture + slicer + verifier (4 agents, not 7) | Re-run without the `quick` argument for the full 7-agent, 4-wave pass |
