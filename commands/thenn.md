---
description: Run, list, or scaffold named AI workflows (.thenn files)
argument-hint: "run <name> | list | new <name>"
allowed-tools: Bash, Read, Glob, Write, Skill, Agent, AskUserQuestion
---

You are handling the `/thenn` command. Arguments: `$ARGUMENTS`

## Step 1 — Parse arguments

Split `$ARGUMENTS` on whitespace. The first token is the subcommand; the second is the flow name; everything after the name is the **remainder**.

- For `run`: remainder is the **input** (e.g. `/thenn run spec-kit add a dark mode toggle` → input=`add a dark mode toggle`)
- For `new`: remainder is the **description** (e.g. `/thenn new my-flow iterative review and fix` → description=`iterative review and fix`)
- For `update`: remainder is the **change description** (e.g. `/thenn update my-flow add a human review step after commit` → change=`add a human review step after commit`)

If no subcommand is provided, output the usage below and stop:

```
Usage:
  /thenn run <name> [input...]           Run a named flow (optional: describe what you want)
  /thenn list                            List available flows (project + personal)
  /thenn new <name> [description]        Scaffold a new .thenn file in .thenn/
  /thenn update <name> [change...]       Edit an existing flow
```

---

## Subcommand: list

Scan for `.thenn` files in these locations:
1. `.thenn/*.thenn` (project-level, current directory)
2. `~/.thenn/*.thenn` (personal)

For each file found, read it and extract the `name` and `description` fields.

Output a formatted list grouped by source, for example:

```
Project flows (.thenn/):
  spec-kit     Spec-driven feature development from a brief
  bug-fix      Structured bug fix with reproduction and verification

Personal flows (~/.thenn/):
  (none)

Run a flow: /thenn run <name>
Bundled starter flows: see the flows/ directory in the thenn plugin.
```

If a directory does not exist or is empty, show `(none)` for that section.

If both are empty, output:
```
No flows found. Create one with: /thenn new <name>
Bundled starter flows are in the thenn plugin's flows/ directory — copy one to .thenn/ to get started.
```

---

## Subcommand: new \<name\>

If no name was provided after `new`, output:
```
Usage: /thenn new <name> [description]
```
Then stop.

Otherwise:

**Load the thenn-builder skill.**

Use the Skill tool to load `thenn-builder` before doing anything else. The skill contains the complete scaffold protocol — follow it to create the flow file, passing `<name>` and `<description>` (if provided).

---

## Subcommand: update \<name\>

If no name was provided after `update`, output:
```
Usage: /thenn update <name> [change description...]
```
Then stop.

Otherwise:

**1. Locate the flow file.**

Check in order:
1. `.thenn/<name>.thenn`
2. `~/.thenn/<name>.thenn`

If not found in either location, output:
```
Flow '<name>' not found. Use /thenn new <name> to create it.
```
Then stop.

**2. Load the thenn-builder skill.**

Use the Skill tool to load `thenn-builder` before doing anything else. The skill contains the complete update protocol — follow it, passing the resolved file path and `<change>` (if provided).

---

## Subcommand: run \<name\>

If no name was provided after `run`, output:
```
Usage: /thenn run <name>
Run /thenn list to see available flows.
```
Then stop.

Otherwise:

**1. Load the thenn-runner skill.**

Use the Skill tool to load `thenn-runner` before doing anything else. The skill contains the complete execution protocol — follow it for all subsequent steps.

**2. Locate the flow file.**

Per the thenn-runner skill discovery protocol, check in order:
1. `.thenn/<name>.thenn`
2. `~/.thenn/<name>.thenn`

If not found in either location, output:
```
Flow '<name>' not found.
```
Then run the list logic (above) to show what flows are available, and stop.

**3. Capture input.**

Read the flow file and check for a top-level `input` field.

| Situation | Action |
|---|---|
| Trailing args provided (e.g. `add dark mode`) | Use the args as the input |
| No args, flow has `input` field | Use AskUserQuestion with the `input` value as the question, single option `["Continue"]`; capture what the user types as the input |
| No args, no `input` field | No input — skip to step 4 |

If input was captured, run `mkdir -p .thenn-state` and write it to `.thenn-state/input.md`.

**4. Execute the flow.**

Follow the thenn-runner skill protocol exactly:
- Run pre-flight checks
- Execute each step in sequence, with input injection if `.thenn-state/input.md` exists
- Report progress and handle failures per the skill's instructions
