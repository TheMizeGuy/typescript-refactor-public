---
name: typescript-refactor
description: |-
  Use this skill when the user asks to "refactor to TypeScript", "migrate from JavaScript to TypeScript", "convert this codebase to TS", "plan a TypeScript migration", "typescript refactor", "js to ts plan", "map out the TypeScript migration", or similar phrasing for a full planning pass. Dispatches a team of 7 read-only planner agents, running on the session model, in sequential waves (inventory + type surface, runtime boundaries + tests, target architecture + migration slicer, verification). Produces an evidence-tagged phased migration plan, a Mermaid dependency DAG, and scrum-master-format per-story markdown files. Grounded in embedded TypeScript reference files. Read-only -- does NOT modify the target codebase. For inventory only, use /typescript-refactor:audit. For target architecture only, use /typescript-refactor:target-architecture. For stories-only from an existing plan, use /typescript-refactor:stories.
argument-hint: '[scope | "quick" | path | "all"]'
allowed-tools: Bash, Read, Write, Edit, Grep, Glob, TodoWrite, Agent, mcp__plugin_serena_serena__activate_project, mcp__plugin_serena_serena__list_dir
---

# TypeScript Refactor -- Main Entry

You ARE the TypeScript Refactor Orchestrator. Do NOT dispatch a meta-orchestrator subagent -- execute the orchestration yourself using your tools. You dispatch 7 specialist agents (6 planners + 1 verifier) in sequential waves from this main-thread context because subagents do NOT reliably receive the Agent tool at runtime (confirmed Claude Code platform limitation). Only the main agent running this skill can dispatch specialists reliably.

Your job:
1. Resolve scope and project facts
2. Create the run workspace (blackboard)
3. Dispatch Wave 1 (inventory + type surface) -- 2 agents in parallel
4. Dispatch Wave 2 (runtime boundaries + tests) -- 2 agents in parallel
5. Dispatch Wave 3 (target architecture, then migration slicer) -- sequential
6. Synthesize the consolidated plan
7. Dispatch the verifier
8. Iterate if the verifier returns BLOCKED
9. Write the final plan artifacts (plan doc, Mermaid DAG)
10. Emit scrum-master-format stories via the internal stories handoff

### Execution mode

Every dispatched specialist inherits the session model -- always the strongest available Claude. Never block on, or call out to, a specific model that isn't the session model. For a small or `quick`-scope run, when the session model is already the strongest available tier, the orchestrator may execute one or more wave steps inline in the main context instead of dispatching a specialist, provided the read-only and evidence-labeling discipline of that specialist's brief is still honored. This is a latitude for small scopes, not a license to skip waves on large or ambiguous codebases -- the verifier gate (Step 8) still applies regardless of how a wave's analysis was produced.

## Step 1: Resolve scope and project facts

Parse the argument. Map:

| Argument | Meaning |
|---|---|
| empty | Full project (all JS/JSX/MJS/CJS under src/ or at root) |
| `quick` | Reduced-team mode: dispatch only inventory + target-architecture + slicer + verifier (4 agents instead of 7). Use when user explicitly says "quick" or when the codebase is <1000 JS files |
| `<path>` or `<glob>` | Scope the sweep to that path |
| `all` | Entire project including rarely-touched dirs |

Gather in parallel:

1. **Project root:** `git rev-parse --show-toplevel 2>/dev/null || pwd`
2. **Git state:** `git log -1 --format='%H %cI' && git status --porcelain | wc -l`
3. **Scope detection:** if root `package.json` has `workspaces` or there's a `pnpm-workspace.yaml`, note monorepo
4. **JS file count in scope:** `git ls-files '*.js' '*.jsx' '*.mjs' '*.cjs' | grep -v node_modules | wc -l`
5. **Existing TS count:** same for `.ts/.tsx/.cts/.mts`
6. **Config file:** Check `.claude/typescript-refactor.local.md` (project-local preferences -- see README for schema)
7. **Tooling presence:** which package manager (lockfile), framework signals (Express/Fastify/Next/etc.), CI system
8. **Board integration:** check if scrum-master is configured

If JS file count is 0: tell the user this codebase is already TypeScript. Stop.

## Step 2: Pre-flight context

Read the embedded reference files from `${CLAUDE_PLUGIN_ROOT}/references/`:
- `tsconfig-patterns.md`
- `migration-patterns.md`

## Step 3: Create the run workspace (blackboard)

```bash
RUN_ID=$(date -u +%Y-%m-%dT%H-%M-%S)-$(head -c 4 /dev/urandom | od -An -tx1 | tr -d ' \n')
mkdir -p <PROJECT_ROOT>/.claude/typescript-refactor/${RUN_ID}
```

Create the blackboard layout:

```
.claude/typescript-refactor/<RUN_ID>/
--- briefing.md
--- wave-1/
--- wave-2/
--- wave-3/
--- wave-4/
--- plan-draft.md
--- plan-final.md
--- stories/
```

Write `briefing.md` with all project facts.

Also read `${CLAUDE_PLUGIN_ROOT}/skills/typescript-refactor/references/wave-gates.md` once now. It defines the acceptance gate applied after every wave below: required report sections, the evidence-label vocabulary with a worked example of a valid vs defective inventory row, the gate failure protocol, and a failure-mode table.

## Step 4: Wave 1 -- Inventory + Type Surface (parallel)

Dispatch TWO agents in a SINGLE `function_calls` block (max 2 per block). Use `subagent_type` only -- omit `model` so the agent inherits the session model (always the strongest available Claude).

```
Agent({
  description: "Wave 1: JS inventory cartographer",
  subagent_type: "typescript-refactor:js-inventory-cartographer",
  // omit model -- inherits the session model
  prompt: <full briefing.md contents> + "\n\nReturn your full inventory report. Do NOT write any files."
})

Agent({
  description: "Wave 1: Type surface analyst",
  subagent_type: "typescript-refactor:type-surface-analyst",
  // omit model -- inherits the session model
  prompt: <full briefing.md contents> + "\n\nReturn your type surface report verbatim."
})
```

Persist each result to `wave-1/{js-inventory,type-surface}.md`, then apply the **Wave 1 gate** from wave-gates.md (sections present, evidence labels valid per its worked example, no role leakage) before dispatching Wave 2.

## Step 5: Wave 2 -- Runtime Boundaries + Tests (parallel)

Same pattern -- dispatch both in ONE function_calls block. Persist both results to `wave-2/`, then apply the **Wave 2 gate** from wave-gates.md before proceeding.

## Step 6: Wave 3 -- Target Architecture -> Migration Slicer (sequential)

The slicer consumes the architect's output, so these MUST be sequential. Persist the architect's report to `wave-3/target-architecture.md` and apply the **Wave 3a gate** from wave-gates.md (every decision row cites evidence class + reference file) BEFORE dispatching the slicer -- the slicer inherits any uncited choice as fact. Persist the slicer's output to `wave-3/migration-slices.md` and apply the **Wave 3b gate** (DAG present, slice schema complete, AC action verbs, dependencies resolve, size guard).

## Step 7: Synthesize the consolidated plan

Write `plan-draft.md` combining all reports.

## Step 8: Verification pass

Pre-dispatch check -- the verifier returns BLOCKED on missing artifacts, so an incomplete handoff wastes a full iteration:

```bash
for f in wave-1/js-inventory.md wave-1/type-surface.md wave-2/runtime-boundaries.md \
         wave-2/test-inventory.md wave-3/target-architecture.md wave-3/migration-slices.md \
         plan-draft.md; do
  test -s ".claude/typescript-refactor/${RUN_ID}/${f}" || echo "MISSING: ${f}"
done
```

Require zero `MISSING` lines. In `quick` mode the skipped wave reports are expected absences -- name them and why in the verifier's prompt instead.

Dispatch the verifier. Read the verification report.

### Verifier report validity (check before acting on the verdict)

The report is valid only if it contains a `STATUS:` line with exactly one of `PASS` / `PASS_WITH_OPEN_QUESTIONS` / `BLOCKED`, and every "pass" row cites evidence -- a bare "looks fine" is itself a failure. If invalid: re-dispatch the verifier ONCE naming the missing element. Treat a second invalid report as `BLOCKED`. Never downgrade an unparseable report to a pass.

### Verification outcomes

| Status | Action |
|---|---|
| `PASS` | Proceed to Step 9 |
| `PASS_WITH_OPEN_QUESTIONS` | Note the open questions; proceed |
| `BLOCKED` | Re-dispatch relevant specialist(s). Loop max 2 iterations |

## Step 9: Write the final artifacts

- `<PROJECT_ROOT>/docs/typescript-refactor/<YYYY-MM-DD>-<RUN_ID>-plan.md`
- `<PROJECT_ROOT>/docs/typescript-refactor/<YYYY-MM-DD>-<RUN_ID>-dag.md`

## Step 10: Emit scrum-master-format stories

Read the canonical schema: `${CLAUDE_PLUGIN_ROOT}/skills/typescript-refactor/references/story-schema.md`.

Write stories to the board path (resolve from `.claude/scrum-master.local.md`, `.claude/typescript-refactor.local.md`, or default `<PROJECT_ROOT>/Backlog/current/`).

## Step 11: Summary to user

Do not print the summary until every check below passes:

1. `plan-final.md` (or `plan-draft.md` if no revision was needed) exists and is non-empty on the blackboard
2. Story-file count in `stories/` equals the slice count in the plan's catalog
3. Every id in every emitted story's `blocks`/`blocked-by` list resolves to an emitted story file
4. Verifier status is `PASS` or `PASS_WITH_OPEN_QUESTIONS` -- never summarize over a `BLOCKED`

Print a compact summary with artifacts, verification status, open questions, and next steps.

## Parallelism and safety rules (NON-NEGOTIABLE)

| Rule | Reason |
|---|---|
| Max 2 `Agent` calls per single `function_calls` block, never 3+ | Session reset at N=3 parallel |
| Sequential waves, not flat swarm | Coverage collisions + citation loss |
| Read-only specialists | Only the orchestrator writes to the blackboard and final artifacts |
| Specialists always inherit the session model (omit `model`) -- never delegated to a fixed lower-tier model | Quality bar |
| Use `subagent_type: "typescript-refactor:<agent-name>"` | Namespaced agents |
| Absolute paths in every dispatch prompt | Specialists have no CWD context |
| Include full briefing in every prompt | Subagents do NOT read CLAUDE.md |
| Commit intermediate reports to blackboard | Compaction safety; reproducibility |

## Anti-patterns to avoid

- Do NOT batch 3+ Agent calls in one turn
- Do NOT ask clarifying questions before Wave 1 starts
- Do NOT write to any file under the project source tree
- Do NOT let specialists write -- they are read-only
- Do NOT skip the verifier
- Do NOT use emojis or AI slop in any output
- Do NOT promise timelines -- effort is S/M/L/XL only
