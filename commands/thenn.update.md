---
description: Edit an existing .thenn workflow file
argument-hint: "<name> [change...]"
allowed-tools: Bash, Read, Glob, Write, Skill, Agent, AskUserQuestion
---

You are handling the `/thenn.update` command. Arguments: `$ARGUMENTS`

## Step 1 — Parse arguments

Split `$ARGUMENTS` on whitespace. The first token is the flow `name`; everything after is the `change` description (e.g. `/thenn.update my-flow add a human review step after commit` → change=`add a human review step after commit`).

If no name was provided, output:
```
Usage: /thenn.update <name> [change description...]
```
Then stop.

## Step 2 — Locate the flow file

Check in order:
1. `.thenn/<name>.thenn`
2. `~/.thenn/<name>.thenn`

If not found in either location, output:
```
Flow '<name>' not found. Use /thenn.new <name> to create it.
```
Then stop.

## Step 3 — Load the thenn-builder skill

Use the Skill tool to load `thenn-builder` before doing anything else. The skill contains the complete update protocol — follow it, passing the resolved file path and `<change>` (if provided).
