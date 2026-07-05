---
name: audit
description: |-
  Use this skill when the user asks to "audit the JavaScript codebase", "inventory this JS project", "what does my JS codebase look like", "typescript refactor audit", "show me the migration baseline", "ts refactor inventory", or wants only the inventory pass without a full migration plan. Dispatches the js-inventory-cartographer and type-surface-analyst agents (Wave 1 only) and returns a consolidated audit report. Read-only. For the full plan with stories, use /typescript-refactor.
argument-hint: '[scope | path]'
allowed-tools: Bash, Read, Write, Edit, Grep, Glob, TodoWrite, Agent, mcp__plugin_serena_serena__activate_project, mcp__plugin_serena_serena__list_dir
---

# TypeScript Refactor -- Audit Only

You are running the inventory-only pass of the TypeScript Refactor plugin. This produces a baseline audit without a target-state or migration plan. Use it when the user wants to understand the current state before committing to a full planning session.

Your job:

1. Resolve scope
2. Create blackboard workspace
3. Dispatch Wave 1 agents (inventory + type surface) in parallel
4. Synthesize a combined audit report
5. Write artifacts and summarize

### Execution mode

The dispatched agents inherit the session model -- always the strongest available Claude. Never block on, or wait for, a specific model that isn't the session model. For a small scope, when the session model is already the strongest available tier, the orchestrator may run the inventory pass inline in the main context instead of dispatching, provided the read-only, evidence-labeled discipline of the specialists' briefs is preserved.

## Step 1: Resolve scope

Parse the argument. Empty = full project, else path/glob scope. Gather:

- `git rev-parse --show-toplevel 2>/dev/null || pwd`
- `git ls-files '*.js' '*.jsx' '*.mjs' '*.cjs' | wc -l`
- `git ls-files '*.ts' '*.tsx' '*.cts' '*.mts' | wc -l`
- Root `package.json` -- package manager, framework, scripts
- `.claude/typescript-refactor.local.md` if present

## Step 2: Create blackboard

```bash
RUN_ID=$(date -u +%Y-%m-%dT%H-%M-%S)-$(head -c 4 /dev/urandom | od -An -tx1 | tr -d ' \n')
mkdir -p <PROJECT_ROOT>/.claude/typescript-refactor/${RUN_ID}/wave-1
```

## Step 3: Dispatch Wave 1

ONE `function_calls` block with both agents:

```
Agent({
  description: "Audit: JS inventory cartographer",
  subagent_type: "typescript-refactor:js-inventory-cartographer",
  // omit model -- inherits the session model
  prompt: "<full briefing + 'Return your inventory report verbatim; I will persist.'>"
})

Agent({
  description: "Audit: Type surface analyst",
  subagent_type: "typescript-refactor:type-surface-analyst",
  // omit model -- inherits the session model
  prompt: "<full briefing>"
})
```

Persist each result to `wave-1/{js-inventory,type-surface}.md`.

Then apply the **Wave 1 gate** from `${CLAUDE_PLUGIN_ROOT}/skills/typescript-refactor/references/wave-gates.md`: required sections present, every fact row carries an evidence label naming a re-checkable artifact (judge against that file's worked example), no role leakage, word caps. On failure, re-dispatch the failing specialist ONCE with the defective rows quoted verbatim; on a second failure, stop and report the exact defect.

## Step 4: Synthesize audit report

Write `<PROJECT_ROOT>/docs/typescript-refactor/<YYYY-MM-DD>-<RUN_ID>-audit.md`.

## Step 5: Summary

Do not print the summary until both wave-1 reports exist on the blackboard and the audit report's at-a-glance table has no blank cells (unresolved values say `OPEN-QUESTION`, never empty).

```
JS Codebase Audit complete

Codebase: <name>
JS files: <N>  |  TS files: <N>  |  Coverage: <N>%

Buckets:
  Ready-to-rename:    <N> files
  Light polish:       <N> files
  JSDoc-first:        <N> files
  Blank slate:        <N> files

Report: <path>
Blackboard: .claude/typescript-refactor/<RUN_ID>/

Next: /typescript-refactor  (produces the full plan + stories using this audit)
```

## Anti-patterns

- Do NOT dispatch target-architect or migration-slicer -- that is the full skill's job
- Do NOT emit stories -- that is /typescript-refactor:stories
- Do NOT batch 3+ Agent calls in one turn
