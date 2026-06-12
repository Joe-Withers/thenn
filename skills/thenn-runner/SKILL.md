---
name: thenn-runner
description: Use this skill when executing a .thenn workflow file. Teaches Claude how to discover, parse, and execute each step in a thenn flow using claude, bash, human, and loop step types. Loaded automatically by the /thenn.run command.
version: 0.3.0
---

# thenn-runner

This skill defines the execution protocol for `.thenn` workflow files. Follow these instructions precisely when running a thenn flow.

---

## File format reference

`.thenn` files are YAML with this top-level structure:

```yaml
name: flow-name           # required
description: "..."        # optional, shown in /thenn.list

steps:
  - id: step-id           # required; used in progress output and error messages
    type: agent           # agent | coordinator | bash | human | loop
    # type-specific fields below
```

**Per step type:**

`type: agent` — runs a command, agent, or prompt in a fresh subagent context:
```yaml
command: name-of-command  # calls /name-of-command slash command
agent: name-of-agent      # spawns a named subagent
prompt: "..."             # standalone: full inline prompt; combined with command/agent: appended as additional context
input: true               # required: true: inject user's run-time description; false: read context from disk
```

`type: coordinator` — same fields as `type: agent`, but the runner processes the step in its own main conversation instead of spawning a subagent. Use this when the step needs to spawn its own subagents, interact with the user mid-flow, or make decisions based on flow state.
```yaml
type: coordinator
command: name-of-command  # runner invokes the slash command in the main conversation
agent: name-of-agent      # runner uses the Agent tool to spawn the named agent
prompt: "..."             # runner processes the prompt in the main conversation
input: true               # required: same as type: agent
```

`prompt` behaviour for both `type: agent` and `type: coordinator`:
- `prompt` alone → used as the full task instruction
- `command` + `prompt` → run the command, append `prompt` as additional context
- `agent` + `prompt` → spawn the agent, append `prompt` as additional context
- `command` + `agent` → use `command`, ignore `agent`, warn once

The difference between the two types is **who does the work**: for `type: agent`, a fresh subagent does it; for `type: coordinator`, the runner (this conversation) does it. See "The context-light coordinator convention" below for the implications.

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
    type: agent
    command: review-code
  - id: fix
    type: agent
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
5. Check whether `.thenn-state/input.md` exists. If it does, read it — this is the user's input for this run and must be injected into every `type: agent` and `type: coordinator` step.
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

## The context-light coordinator convention

A `type: coordinator` step is run by **you** — the runner — in your own main conversation, not delegated to a subagent. This is the only way the step can spawn its own subagents, call `AskUserQuestion` mid-flow, or make decisions based on the flow's overall state, because subagents cannot spawn further subagents and have no view of the flow outside their own prompt.

When you (the runner) execute a `type: coordinator` step, you are acting as a **context-light coordinator** for that step. The convention:

**Do:**
- **Delegate heavy work to subagents.** Use the Agent tool. Your main context grows with each step — a coordinator's job is to dispatch, not to do.
- **Interact with the user via `AskUserQuestion`** when the step needs a clarification that only the human can give.
- **Write outputs to disk** so downstream steps can read them. Filesystem-as-context-bus still applies.
- **Return a brief one-line summary** when the step is done, then move on. The user can read your progress in the terminal — you don't need to recapitulate.

**Don't:**
- **Do not try to execute future flow steps.** The runner (you) is the orchestrator; you will move to step N+1 after this one. Do not pre-empt.
- **Do not modify the flow file or runner state.** Only write to the working directory.
- **Do not get lost in the step.** Always know "this is step X of N in flow Y; when done, move to X+1." A long step must not make you forget where you are in the flow.

### The return protocol

When the step's work is done, you must explicitly return to the flow:

1. Announce completion: `[thenn] Step X/N: [step-id] done`
2. Move on to the next step in the flow YAML, applying the normal execution rules (input injection, progress format, etc.).

This is what stops a long-running `type: coordinator` step from making the orchestrator "lose the thread." The user is reading your progress in their terminal — they see you announce each step and each completion. As long as you keep the announcements coming, the flow stays inspectable.

### When to use `type: coordinator` vs `type: agent`

Use `type: agent` (spawn subagent) when:
- The step is a single, self-contained piece of work (write a file, run a command, read-and-summarise).
- You do not need the subagent to spawn more subagents or to interact with the user mid-step.
- The convention is "context-light" — the subagent gets a fresh context and a focused task.

Use `type: coordinator` (runner does the work) when:
- The step needs to **fan out** to multiple subagents in parallel (a coordinator dispatching work).
- The step needs to **ask the user** something mid-step (e.g., a clarification that determines the rest of the flow).
- The step needs to **make a decision** based on which steps remain, or based on the overall flow state, in a way that a subagent with a fresh context could not.
- The step is a **single complex action** that you (the runner) can do more cheaply in the main conversation than by spawning a subagent.

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

### type: agent — with `command`

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

### type: agent — with `agent`

Spawn a subagent using the Agent tool:
- `subagent_type`: use the `agent` value if it matches a known subagent type; otherwise use `"general-purpose"` and reference the agent name in the prompt
- `description`: name the agent and its role in the flow
- `prompt`: describe the task clearly, including what files to read and what output is expected, followed by any `prompt` value from the step as an "Additional context:" block, then the input block if `input: true`

Wait for the subagent to complete before proceeding.

---

### type: agent — with `prompt`

Spawn a subagent using the Agent tool:
- `subagent_type: "general-purpose"`
- `description`: a short summary of the inline prompt
- `prompt`: the exact value from the `prompt` field in the YAML (append input block if `input: true` — see below)

Wait for the subagent to complete before proceeding.

---

### type: coordinator

`type: coordinator` accepts the same fields as `type: agent` (`command:`, `agent:`, `prompt:`, `input:`), but the runner — not a subagent — does the work in this main conversation. This is the only way the step can spawn its own subagents, interact with the user mid-step, or read the flow's overall state.

**How to process the step:**

1. Announce: `[thenn] Step X/N: [step-id] <brief action description>`
2. Read the step's task. The flavour depends on what other fields the step has:
   - `prompt: "..."` → that is your task. Process it now, in the main conversation.
   - `command: name` → resolve the slash command (same path rules as the subagent case) and follow its instructions in the main conversation. If the command's file is a skill (`SKILL.md`), use the Skill tool to load it.
   - `agent: name` → spawn the named agent using the Agent tool, as a sub-subagent. This is the one case where `type: coordinator` does involve a subagent — but the runner, not the step's author, decides to spawn it, and the runner can spawn multiple or none.
3. Combine with the step's other fields as follows:
   - If `prompt` is set together with `command` or `agent`, treat the `prompt` value as "additional context" and use it to steer the work.
   - If `input: true`, read `.thenn-state/input.md` (if it exists) and incorporate the user's request into your work for this step.
4. Use any tools you have access to. The point of `type: coordinator` is that you can now:
   - Call the **Agent tool** to spawn sub-subagents (a subagent cannot do this)
   - Call **AskUserQuestion** to interact with the user mid-step (a subagent can do this too, but `type: coordinator` lets the runner do it without losing the flow thread)
   - Make decisions based on what other steps have run, or on flow state
5. When the step's work is done, announce: `[thenn] Step X/N: [step-id] done` and proceed to the next step in the flow YAML.
6. If something goes wrong that you cannot recover from, log `[thenn] Step X/N: [step-id] FAILED — <reason>` and stop the flow (use the same error-handling rules as the other step types).

**Conventions for `type: coordinator` steps** (see "The context-light coordinator convention" above for the full set):
- Be a coordinator, not a doer. Delegate heavy work to subagents.
- Write outputs to disk.
- Return a brief summary, then move to the next step. Do not try to run future flow steps.

---

### Input injection (`input: true | false`)

Every `type: agent` and `type: coordinator` step must declare `input: true` or `input: false`. This makes the intent explicit — which steps receive the user's run-time description and which rely solely on files written by earlier steps.

- `input: true` — for `type: agent` steps, append the contents of `.thenn-state/input.md` to the subagent prompt:
  ```
  User's request for this flow run:
  <contents of .thenn-state/input.md>
  ```
  For `type: coordinator` steps, you are the executor — read `.thenn-state/input.md` yourself and incorporate the user's request into the work for this step.
- `input: false` — do not append any input; the step reads its context from disk (or, for `type: coordinator`, you read from disk and proceed without the user's original request)

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
   - If a sub-step halts the flow (bash failure with on_failure: stop, or agent/coordinator failure), halt immediately

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
| `type: agent` + `command` not found | Stop, name the missing command |
| `type: agent` subagent reports failure | Stop, report the error output |
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
| `type: coordinator` and `command` not found | Stop, name the missing command (same as `type: agent`) |
| `type: coordinator` cannot complete | Stop, report the error (same as `type: agent`) |
