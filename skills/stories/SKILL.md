---
name: stories
description: |-
  Use this skill when the user asks to "create stories from the typescript refactor plan", "emit stories for ts migration", "scrum-master stories from migration plan", "ts refactor stories", "hand off the typescript plan to scrum master", or has an existing migration plan (or migration-slices report) on disk and wants per-story scrum-master-format markdown files generated. Reads a plan file (or the most recent blackboard plan) and emits one story file per slice in the exact scrum-master schema, with dependency rewrites, AC validation, and Mermaid DAG. Read-mostly -- writes stories to the scrum-master board path. For a full plan from scratch, use /typescript-refactor.
argument-hint: '[plan-file-path | "latest" | --board=<path>]'
allowed-tools: Bash, Read, Write, Edit, Grep, Glob, TodoWrite, mcp__plugin_serena_serena__activate_project
---

# TypeScript Refactor -- Stories Handoff

You are running the stories handoff. You take an existing migration plan (or the slicer's per-slice catalog) and write one scrum-master-format story file per slice.

Your job:

1. Locate the plan
2. Resolve the board path
3. Validate the plan's slice catalog
4. Write one story file per slice
5. Write the Mermaid DAG
6. Summarize

## Step 1: Locate the plan

| Argument | Action |
|---|---|
| empty or `latest` | Find newest `.claude/typescript-refactor/*/plan-final.md` |
| `<path>` | Use the path verbatim |
| `--board=<path>` | Use the path AND override the board target |

If no plan exists: tell the user to run `/typescript-refactor` first. Stop.

## Step 2: Resolve the board path

1. Inline `--board=<path>` argument
2. `.claude/scrum-master.local.md` `board_path:` field
3. `.claude/typescript-refactor.local.md` `scrum_master_board_path:` field
4. Auto-detect: glob `**/Backlog/current/`
5. Default: `<PROJECT_ROOT>/Backlog/current/`

## Step 3: Validate the slice catalog

Parse the plan's slice catalog. Verify required fields exist. FAIL on missing title, scope.include, acceptance with prose-only AC, or missing rollback.

## Step 4: Scan existing board for ID collisions

On collision, append `A` suffix and rewrite all dependency references.

## Step 5: Emit one story file per slice

Read the canonical schema: `${CLAUDE_PLUGIN_ROOT}/skills/typescript-refactor/references/story-schema.md`. Instantiate it for each slice.

## Step 6: Write Mermaid DAG

Generate `<BOARD_PATH>/../board-dag.md` with color-coded nodes per epic.

## Step 7: Summary

```
Stories handoff complete

Source plan:    <path>
Board path:     <abs>
Stories written: <N>

By epic:
  Foundation (F):   <n>
  Boundaries (B):   <n>
  Renames (R):      <n>
  Framework (FW):   <n>
  Cleanup (C):      <n>

Next:
  /scrum-master status     # see populated board
  /scrum-master deps       # render the DAG
```

## Anti-patterns

- Do NOT modify the source plan
- Do NOT skip AC validation
- Do NOT emit stories with empty `scope.include`
- Do NOT emit stories without rollback
- Do NOT silently drop dependencies when ID collisions force renames
