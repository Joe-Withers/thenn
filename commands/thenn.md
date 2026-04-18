---
description: Run, list, or scaffold named AI workflows (.thenn files)
argument-hint: "run <name> | list | new <name>"
allowed-tools: Bash, Read, Glob, Write, Skill, Agent, AskUserQuestion
---

You are handling the `/thenn` command. Arguments: `$ARGUMENTS`

## Step 1 — Parse arguments

Split `$ARGUMENTS` on whitespace. The first token is the subcommand; the second is the flow name (for `run` and `new`); everything after the flow name is the **input** (for `run`).

For example: `/thenn run spec-kit add a dark mode toggle` → subcommand=`run`, name=`spec-kit`, input=`add a dark mode toggle`

If no subcommand is provided, output the usage below and stop:

```
Usage:
  /thenn run <name> [input...]    Run a named flow (optional: describe what you want)
  /thenn list                     List available flows (project + personal)
  /thenn new <name>               Scaffold a new .thenn file in .thenn/
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

1. Run `mkdir -p .thenn` to ensure the directory exists.
2. Check if `.thenn/<name>.thenn` already exists. If it does, use AskUserQuestion to confirm: "`.thenn/<name>.thenn` already exists. Overwrite?" with options `["Yes, overwrite", "Cancel"]`. If cancelled, stop.
3. Write the following scaffold to `.thenn/<name>.thenn`, substituting `<name>` for the actual name:

```yaml
name: <name>
description: Describe what this flow does
input: "What do you want to build?"   # optional: ask this if no args given at run time

steps:
  # Run a slash command in a fresh context
  - id: step-one
    type: claude
    command: your-command-name   # must exist in .claude/commands/

  # Pause for human review
  - id: review
    type: human
    message: "Review the output, then press Continue"

  # Run a shell command
  - id: commit
    type: bash
    run: git add . && git commit -m "chore: from thenn flow"
    on_failure: stop   # stop | continue | human
```

4. Confirm to the user:
   ```
   Created .thenn/<name>.thenn
   Edit it to define your steps, then run it with: /thenn run <name>
   ```

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
