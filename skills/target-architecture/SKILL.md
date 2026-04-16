---
name: target-architecture
description: |-
  Use this skill when the user asks to "design the TypeScript target architecture", "what should my TS target state be", "pick TS framework and tools", "tsconfig recommendation", "typescript refactor target architecture", "decide TS stack for migration", or wants the architecture decision matrix without the full migration plan. Dispatches the typescript-target-architect agent backed by the embedded TypeScript reference files. Returns evidence-tagged decisions with citations. Read-only. For the full migration plan, use /typescript-refactor.
argument-hint: '[scope | path | --constraints="<inline preferences>"]'
allowed-tools: Bash, Read, Write, Edit, Grep, Glob, TodoWrite, Agent, mcp__plugin_serena_serena__activate_project
---

# TypeScript Refactor -- Target Architecture Only

You are running the target-architecture-only pass. This produces the cutting-edge TypeScript target state (tsconfig, framework, tools, lint config, CI gates) without the migration sequencing.

Useful when the user wants to compare options before committing to a full plan, or has an existing partial migration and needs to lock in the target.

Your job:

1. Resolve scope and gather minimum context
2. Optionally load any existing audit on the blackboard
3. Dispatch the typescript-target-architect agent
4. Persist the output and summarize

## Step 1: Scope and context

```bash
git rev-parse --show-toplevel 2>/dev/null || pwd
git ls-files '*.js' '*.jsx' '*.mjs' '*.cjs' | wc -l
git ls-files '*.ts' '*.tsx' | wc -l
cat package.json | head -50
```

Read `.claude/typescript-refactor.local.md` if present.

## Step 2: Look for existing audit on blackboard

```bash
ls .claude/typescript-refactor/*/wave-1/js-inventory.md 2>/dev/null | tail -1
ls .claude/typescript-refactor/*/wave-2/runtime-boundaries.md 2>/dev/null | tail -1
```

## Step 3: Generate run ID and create blackboard

```bash
RUN_ID=$(date -u +%Y-%m-%dT%H-%M-%S)-$(head -c 4 /dev/urandom | od -An -tx1 | tr -d ' \n')
mkdir -p <PROJECT_ROOT>/.claude/typescript-refactor/${RUN_ID}/wave-3
```

## Step 4: Dispatch typescript-target-architect

```
Agent({
  description: "Target architecture decision",
  subagent_type: "typescript-refactor:typescript-target-architect",
  model: "opus",
  prompt: "<full briefing including PROJECT_ROOT, RUN_ID, USER_CONSTRAINTS, paths to any existing reports>"
})
```

Persist to `wave-3/target-architecture.md`.

## Step 5: Write artifact

`<PROJECT_ROOT>/docs/typescript-refactor/<YYYY-MM-DD>-<RUN_ID>-target-architecture.md`

## Step 6: Summary

```
TypeScript Target Architecture -- decisions ready

Headline choices:
  TS version:    <version>
  Module system: <NodeNext|ESNext|Bundler>
  HTTP server:   <Fastify|Hono|Nest|Express|none>
  Validation:    <Zod|Valibot|Arktype|Typebox>
  Data layer:    <Drizzle|Prisma|Kysely|driver>
  Test runner:   <Vitest|Jest|Bun test|Node --test>
  Lint:          <typescript-eslint|biome>

Report: <path>

Next:
  /typescript-refactor    # produce the migration plan + stories using this target
  /typescript-refactor:audit  # if you also want the inventory pass
```

## Anti-patterns

- Do NOT propose the migration sequence -- that's the slicer's job
- Do NOT recommend without reference file citation
- Do NOT skip the user's constraints from `.local.md` or the inline flag
