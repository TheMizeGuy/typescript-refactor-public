# Wave gates -- acceptance criteria applied after every wave

Read by the `typescript-refactor` orchestrator (all waves) and the `audit` orchestrator (Wave 1 gate only). Apply the matching gate AFTER persisting a wave's reports to the blackboard and BEFORE dispatching the next wave. A report that fails its gate gets ONE re-dispatch of the failing specialist with the defective rows quoted verbatim; a second failure stops the run with the exact defect reported to the user. Never proceed to a later wave over a failed gate -- downstream specialists compound upstream defects.

## Evidence labels (the vocabulary every gate checks)

| Label | Meaning | Emitted by |
|---|---|---|
| `SOURCE` | Direct file observation -- a concrete path or command output is named | all specialists |
| `BUILD` | Derived from build metadata (lockfile, script output, CI config) | inventory, architect |
| `RUNTIME` | Observed runtime behavior | boundary analyst |
| `DOC` | Documented elsewhere (README, ADR, code comment) | all |
| `INFERRED` | Reasoned from 2+ signals, none individually decisive -- signals must be listed | all |
| `TARGET-ASSUMPTION` | A target-state choice, not an observation of the codebase | architect, slicer |
| `OPEN-QUESTION` | Could not resolve; user must decide | all |

## Worked example: judging an evidence-labeled inventory row

Correct row -- the label names a re-checkable artifact:

| Field | Value | Evidence |
|---|---|---|
| Package manager | pnpm 9 | `SOURCE: pnpm-lock.yaml (lockfileVersion '9.0')` |

Accept: label is from the approved set, and the evidence names a concrete file another agent could open to confirm. `BUILD: pnpm ls --depth=-1 -> 14 packages` is equally acceptable -- a command plus its observed output.

Defective row A -- label present, but the evidence is a conclusion, not an artifact:

| Field | Value | Evidence |
|---|---|---|
| Package manager | pnpm | `SOURCE: it is a modern monorepo` |

Reject. "It is a modern monorepo" cannot be re-checked. Re-dispatch instruction to quote: "Row 'Package manager': replace the evidence with the observed file path or command output; or relabel `INFERRED` and list the signals; or relabel `OPEN-QUESTION`."

Defective row B -- no label at all:

| Field | Value | Evidence |
|---|---|---|
| Test runner | Vitest | fast and modern |

Reject. No label from the set; "fast and modern" is opinion.

## Wave 1 gate

| # | Check | How (mechanical) |
|---|---|---|
| 1 | Both reports persisted and non-trivial | `wave-1/js-inventory.md` and `wave-1/type-surface.md` exist, each >80 lines |
| 2 | Inventory sections present | grep js-inventory.md for `## Migration risk signals`, `## Open questions`, `## Sources examined`, `## Confidence` -- all four hit |
| 3 | Type-surface sections present | grep type-surface.md for `## Day-one migration buckets`, `## Escape hatch inventory`, `## Sources examined`, `## Confidence` |
| 4 | Evidence discipline | spot-check 5 fact rows per report against the worked example above -- every one carries an approved label naming a re-checkable artifact |
| 5 | No role leakage | neither report proposes a target tsconfig, migration slices, or waves (Wave 3's job) |

## Wave 2 gate

| # | Check | How (mechanical) |
|---|---|---|
| 1 | Both reports persisted and non-trivial | `wave-2/runtime-boundaries.md` and `wave-2/test-inventory.md` exist |
| 2 | Boundary sections present | grep runtime-boundaries.md for `## Recommended Zod placement`, `## Dynamic code / eval zones`, `## Open questions`, `## Confidence` |
| 3 | Test sections present | grep test-inventory.md for `## AI-cheat red flags`, `## Parity-harness candidates`, `## Open questions`, `## Confidence` |
| 4 | Evidence discipline | spot-check 5 catalog rows per report as in the worked example |
| 5 | Catalogs are countable | boundary catalogs report counts plus example paths, not vague prose ("many routes") |

## Wave 3a gate (target-architecture.md)

| # | Check | How (mechanical) |
|---|---|---|
| 1 | Report persisted | `wave-3/target-architecture.md` exists |
| 2 | Reference reads cited | every recommendation cites a reference file from `${CLAUDE_PLUGIN_ROOT}/references/` per the architect's "Reference files" table |
| 3 | Every decision cited | each decision-matrix row carries an evidence class (`SOURCE`/`BUILD`/`INFERRED`/`TARGET-ASSUMPTION`) -- an uncited recommendation is opinion and fails the row |
| 4 | Grounded in prior waves | choices reference Wave 1-2 facts (e.g. the framework decision cites the inventory's framework row), not generic preference |
| 5 | No role leakage | no slice ordering or wave plan (slicer's job) |

## Wave 3b gate (migration-slices.md)

| # | Check | How (mechanical) |
|---|---|---|
| 1 | Report persisted | `wave-3/migration-slices.md` exists |
| 2 | DAG present | contains a ` ```mermaid ` fence with `graph TD` |
| 3 | Slice schema complete | every slice has `id`, `acceptance`, `cutover`, `rollback` populated -- no empty rollback, never "fix forward" |
| 4 | AC quality | every acceptance criterion contains an action verb from story-schema.md's approved set (`exists`, `passes`, `exit 0`, `tsgo --noEmit`, `grep`, ...) |
| 5 | Dependencies resolve | every id in `blocks`/`blocked-by` names a slice defined in the same catalog |
| 6 | Size guard | no slice exceeds the story-schema.md size guard (>5 AC or >10 scope files) |

## Gate failure protocol

1. Quote the failing gate row AND the defective report excerpt verbatim.
2. Re-dispatch ONLY the failing specialist, once, with both quotes plus the specific fix instruction (as in the worked example's re-dispatch line).
3. On second failure: stop. Report to the user the wave, the gate row, and the verbatim defect. Do not proceed, do not patch the report yourself -- the specialist's evidence trail is the product.

## Failure modes

| Symptom | Likely cause | Fix |
|---|---|---|
| Agent errors immediately or returns empty | `subagent_type` not plugin-qualified | Use `typescript-refactor:<agent-name>` exactly |
| Report contradicts obvious project facts (wrong package manager, wrong framework) | Briefing placeholder like `<Briefing from Step 3>` was echoed literally instead of inlined | Re-dispatch with the full briefing.md contents pasted into the prompt |
| Report is rich prose with no tables or labels | Specialist ignored its report template | Re-dispatch quoting the agent file's report template section |
| Verifier returns BLOCKED citing missing artifacts | A wave report was never persisted | Write the tool result to the blackboard, read it back, re-dispatch the verifier |
| Two specialists disagree on a count (routes, files, tests) | Different scope interpretation | Record both counts in plan-draft.md; the verifier's consistency audit adjudicates -- never silently pick one |
| Same specialist fails the same gate twice | Scope too large or codebase pattern outside the brief | Stop and surface to the user; suggest narrowing scope |
