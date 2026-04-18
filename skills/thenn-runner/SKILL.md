---
name: thenn-runner
description: Use this skill when executing a .thenn workflow file. Teaches Claude how to discover, parse, and execute each step in a thenn flow using claude, bash, human, and loop step types. Loaded automatically by the /thenn run command.
version: 0.1.0
---

# thenn-runner

This skill defines the execution protocol for `.thenn` workflow files. Follow these instructions precisely when running a thenn flow.

---

## File format reference

`.thenn` files are YAML with this top-level structure:

```yaml
name: flow-name           # required
description: "..."        # optional, shown in /thenn list

steps:
  - id: step-id           # required; used in progress output and error messages
    type: claude          # claude | bash | human | loop
    # type-specific fields below
```

**Per step type:**

`type: claude` — runs a command or agent in a fresh subagent context:
```yaml
command: name-of-command  # calls /name-of-command slash command
agent: name-of-agent      # spawns a named subagent
prompt: "..."             # standalone: full inline prompt; combined with command/agent: appended as additional context
```

`prompt` behaviour:
- `prompt` alone → used as the full subagent instruction
- `command` + `prompt` → run the command, append `prompt` as additional context
- `agent` + `prompt` → spawn the agent, append `prompt` as additional context
- `command` + `agent` → use `command`, ignore `agent`, warn once

`type: bash` — runs a shell command:
```yaml
run: "shell command here"
on_failure: stop          # stop | continue | human (default: stop)
```

`type: human` — pauses for user review:
```yaml
message: "Instructions for the human"
```

`type: loop` — repeats sub-steps until an exit condition is met:
```yaml
max_iterations: 5          # required, integer >= 1
until:                     # required, bash check; exit code 0 = done
  type: bash
  run: grep -q "^status: pass" docs/review.md
steps:                     # required, sub-steps to repeat
  - id: review
    type: claude
    command: review-code
  - id: fix
    type: claude
    agent: implementer
```

---

## Discovering a flow file

Check these locations in order, stopping at the first match:

1. `.thenn/<name>.thenn` — project-level (current working directory)
2. `~/.thenn/<name>.thenn` — personal, available in all projects

Use Bash to check:
```bash
ls ".thenn/<name>.thenn" 2>/dev/null || ls "$HOME/.thenn/<name>.thenn" 2>/dev/null
```

If neither exists, list available flows from both locations and stop with a clear message naming which flow was not found.

---

## Before executing any steps

1. Read the flow file with the Read tool
2. Confirm it has a `name` field and a `steps` field. If either is missing, abort: "Flow file is missing required field: `<field>`."
3. If `steps` is empty or has zero items, warn: "Flow '<name>' has no steps." and stop.
4. Count total steps (N) for progress reporting.
5. Check whether `.thenn-state/input.md` exists. If it does, read it — this is the user's input for this run and must be injected into every `type: claude` step.
6. Announce the start: `[thenn] Starting flow "<name>" — N steps`

---

## Progress format

Announce each step before executing it:
```
[thenn] Step X/N: [step-id] <brief action description>
```

After each step completes successfully:
```
[thenn] Step X/N: [step-id] done
```

On failure:
```
[thenn] Step X/N: [step-id] FAILED — <reason>
```

---

## Executing each step type

### type: bash

Run the value of `run` using the Bash tool.

**on_failure behavior:**

| Value | Behavior |
|---|---|
| `stop` (default) | Show the error output. Halt the flow immediately. |
| `continue` | Log `[thenn] Warning: step [id] failed, continuing` and proceed to the next step. |
| `human` | Show the error output, then ask using AskUserQuestion: "Step [id] failed. What would you like to do?" with options: `["Retry", "Skip this step", "Abort flow"]`. Retry re-runs the same command. Skip proceeds to the next step. Abort halts the flow. |

---

### type: claude — with `command`

1. Verify the command file exists. Resolve the file path from the command name:
   - Colons indicate a subdirectory: `gsd:spec-phase` → `gsd/spec-phase.md`
   - No colon: use name as-is: `speckit.specify` → `speckit.specify.md`

   Check both project and personal locations:
   ```bash
   ls ".claude/commands/<path>.md" ~/.claude/commands/<path>.md 2>/dev/null | head -1
   ```
2. If not found: stop the flow. Output: `[thenn] Step [id]: command '/<name>' not found in .claude/commands/ or ~/.claude/commands/. Stopping.`
3. If found: spawn a subagent using the Agent tool:
   - `subagent_type: "general-purpose"`
   - `description: "Run the /<name> command"`
   - `prompt: "Run the /<name> slash command and complete the task it describes."` followed by any `prompt` value from the step as an "Additional context:" block, then the input block if `input: true`
4. Wait for the subagent to complete before moving to the next step.
5. If the subagent reports it could not complete the task, stop the flow and report the error.

---

### type: claude — with `agent`

Spawn a subagent using the Agent tool:
- `subagent_type`: use the `agent` value if it matches a known subagent type; otherwise use `"general-purpose"` and reference the agent name in the prompt
- `description`: name the agent and its role in the flow
- `prompt`: describe the task clearly, including what files to read and what output is expected, followed by any `prompt` value from the step as an "Additional context:" block, then the input block if `input: true`

Wait for the subagent to complete before proceeding.

---

### type: claude — with `prompt`

Spawn a subagent using the Agent tool:
- `subagent_type: "general-purpose"`
- `description`: a short summary of the inline prompt
- `prompt`: the exact value from the `prompt` field in the YAML (append input block if `input: true` — see below)

Wait for the subagent to complete before proceeding.

---

### Input injection (`input: true | false`)

Every `type: claude` step must declare `input: true` or `input: false`. This makes the intent explicit — which steps receive the user's run-time description and which rely solely on files written by earlier steps.

- `input: true` — append the contents of `.thenn-state/input.md` to the subagent prompt:
  ```
  User's request for this flow run:
  <contents of .thenn-state/input.md>
  ```
- `input: false` — do not append any input; the step reads its context from disk

When generating or authoring flows, set `input: true` on any step that needs the user's original description to do its job correctly — not just the first step. Ask: "Could this step do its job using only files on disk, without knowing what the user originally asked for?" If no, use `input: true`. Steps that purely execute against an already-written plan or spec can use `input: false`.

If `input: true` is set but `.thenn-state/input.md` does not exist (no input was provided at run time), proceed without the block — don't error.

---

### type: human

1. Output the `message` value to the user.
2. Use AskUserQuestion:
   - Question: `"Ready to continue?"`
   - Options: `["Continue"]`
3. Wait for the user to respond before proceeding to the next step.

---

### type: loop

**Pre-flight:** verify `max_iterations` and `until` are both present. If either is missing, abort the flow with a clear error. If `steps` is empty, warn and skip the loop (treat as a no-op).

**Execution algorithm:**

```
1. Evaluate the `until` bash command
   - Exit 0 → condition already met; output "[thenn] Loop [id]: condition already met, skipping" and move to next flow step
   - Non-zero or error → proceed

2. Announce: "[thenn] Loop [id] — starting iteration 1/<max_iterations>"

3. Run each sub-step in sequence using the same rules as top-level steps
   - Progress uses scoped ids: "[thenn]   Step X/Y: [loop-id/step-id] ..."
   - If a sub-step halts the flow (bash failure with on_failure: stop, or claude failure), halt immediately

4. Re-evaluate the `until` bash command
   - Exit 0 → output "[thenn] Loop [id]: converged after N iteration(s)" and move to next flow step
   - Non-zero or error → increment iteration counter

5. If iteration counter >= max_iterations:
   output "[thenn] Loop [id]: did not converge after <max_iterations> iteration(s). Stopping flow." and halt

6. Go to step 2
```

**`until` error handling:** if the `until` command itself errors (cannot be run), treat it as "condition not yet met" — do not halt the flow over a failing exit condition check.

**`max_iterations: 0`:** treat as 1 and warn: "`max_iterations` must be >= 1, using 1."

---

## Flow completion

On success (all steps completed):
```
[thenn] Flow "<name>" complete. All N steps succeeded.
```

On early stop:
```
[thenn] Flow "<name>" stopped at step [id] (Step X/N). <reason>
```

---

## Full error handling reference

| Situation | Behavior |
|---|---|
| Flow file not found | List available flows from `.thenn/` and `~/.thenn/`, then stop |
| File missing `name` or `steps` | Abort with parse error naming the missing field |
| `steps` is empty | Warn "flow has no steps", stop |
| Unknown `type` value | Warn `[thenn] Step [id]: unknown type '<value>', skipping` and continue |
| `type: claude` + `command` not found | Stop, name the missing command |
| `type: claude` subagent reports failure | Stop, report the error output |
| `type: bash` fails, no `on_failure` set | Stop (default behavior) |
| `type: bash` fails, `on_failure: continue` | Warn and proceed |
| `type: bash` fails, `on_failure: human` | Ask: retry / skip / abort |
| Step missing `id` | Use `step-N` as fallback (e.g. `step-3`) — don't fail |
| `command` + `agent` both set | Use `command`, ignore `agent`, warn once |
| `command` or `agent` + `prompt` | Run command/agent with `prompt` appended as additional context |
| `type: loop` missing `max_iterations` | Abort: "`loop` step requires `max_iterations`" |
| `type: loop` missing `until` | Abort: "`loop` step requires `until`" |
| `type: loop` `steps` empty | Warn and skip — treat as no-op |
| `type: loop` `max_iterations: 0` | Treat as 1, warn |
| `type: loop` `until` command errors | Treat as "condition not met", continue looping |
| `type: loop` sub-step halts flow | Halt the entire flow immediately |
| `type: loop` exceeds `max_iterations` | Halt flow with "did not converge" message |
| `type: loop` condition met on first check | Skip all iterations |
