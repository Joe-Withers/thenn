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

If no subcommand is provided, output the usage below and stop:

```
Usage:
  /thenn run <name> [input...]    Run a named flow (optional: describe what you want)
  /thenn list                     List available flows (project + personal)
  /thenn new <name> [description] Scaffold a new .thenn file in .thenn/
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
3. If a description was provided, use it to generate a complete, meaningful flow rather than the generic scaffold. Design the steps based on the described workflow, then apply this rule to every `type: claude` step:

   **Deciding `input: true` vs `input: false`:**
   Set `input: true` on any step that needs the user's original run-time description to do its job — not just the first step. Ask: "Could this step do its job correctly using only files on disk, without knowing what the user originally asked for?" If no, set `input: true`.

   Common pattern: early steps that establish context (reproduce, plan, specify) often need `input: true` even if a previous step already ran, because the files written so far may not capture the full human intent. Later steps that purely execute against a written plan or spec can use `input: false`.

4. Write the flow to `.thenn/<name>.thenn`, substituting `<name>` for the actual name and `<description>` for the remainder text if provided, or `Describe what this flow does` if not. If no description was provided, use the generic scaffold below:

```yaml
name: <name>
description: <description>
input: "What do you want to build?"   # optional: ask this if no args given at run time

steps:
  # Run a slash command in a fresh context
  - id: step-one
    type: claude
    input: true                  # true: inject user's run-time description; false: read context from disk
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
