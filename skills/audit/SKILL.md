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
  model: "opus",
  prompt: "<full briefing + 'Return your inventory report verbatim; I will persist.'>"
})

Agent({
  description: "Audit: Type surface analyst",
  subagent_type: "typescript-refactor:type-surface-analyst",
  model: "opus",
  prompt: "<full briefing>"
})
```

Persist each result to `wave-1/{js-inventory,type-surface}.md`.

## Step 4: Synthesize audit report

Write `<PROJECT_ROOT>/docs/typescript-refactor/<YYYY-MM-DD>-<RUN_ID>-audit.md`.

## Step 5: Summary

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
