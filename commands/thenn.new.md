---
description: Scaffold a new .thenn workflow file
argument-hint: "<name> [description]"
allowed-tools: Bash, Read, Glob, Write, Skill, Agent, AskUserQuestion
---

You are handling the `/thenn.new` command. Arguments: `$ARGUMENTS`

## Step 1 — Parse arguments

Split `$ARGUMENTS` on whitespace. The first token is the flow `name`; everything after is the `description`.

If no name was provided, output:
```
Usage: /thenn.new <name> [description]
```
Then stop.

## Step 2 — Load the thenn-builder skill

Use the Skill tool to load `thenn-builder` before doing anything else. The skill contains the complete scaffold protocol — follow it to create the flow file, passing `<name>` and `<description>` (if provided).
