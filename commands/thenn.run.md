---
description: Run a named .thenn flow
argument-hint: "<name> [input...]"
allowed-tools: Bash, Read, Glob, Write, Skill, Agent, AskUserQuestion
---

You are handling the `/thenn.run` command. Arguments: `$ARGUMENTS`

## Step 1 — Parse arguments

Split `$ARGUMENTS` on whitespace. The first token is the flow `name`; everything after is the `input` (e.g. `/thenn.run spec-kit add a dark mode toggle` → input=`add a dark mode toggle`).

If no name was provided, output:
```
Usage: /thenn.run <name>
Run /thenn.list to see available flows.
```
Then stop.

## Step 2 — Load the thenn-runner skill

Use the Skill tool to load `thenn-runner` before doing anything else. The skill contains the complete execution protocol — follow it for all subsequent steps.

## Step 3 — Locate the flow file

Per the thenn-runner skill discovery protocol, check in order:
1. `.thenn/<name>.thenn`
2. `~/.thenn/<name>.thenn`

If not found in either location, output:
```
Flow '<name>' not found.
```
Then run the list logic (see `/thenn.list`) to show what flows are available, and stop.

## Step 4 — Capture input

Read the flow file and check for a top-level `input` field.

| Situation | Action |
|---|---|
| Trailing args provided (e.g. `add dark mode`) | Use the args as the input |
| No args, flow has `input` field | Use AskUserQuestion with the `input` value as the question, single option `["Continue"]`; capture what the user types as the input |
| No args, no `input` field | No input — skip to step 5 |

If input was captured, run `mkdir -p .thenn-state` and write it to `.thenn-state/input.md`.

## Step 5 — Execute the flow

Follow the thenn-runner skill protocol exactly:
- Run pre-flight checks
- Execute each step in sequence, with input injection if `.thenn-state/input.md` exists
- Report progress and handle failures per the skill's instructions
