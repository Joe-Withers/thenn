---
name: thenn-builder
description: Use this skill when scaffolding or updating a .thenn workflow file. Teaches Claude how to generate and modify flows, apply the input injection rules, and write the file to .thenn/. Loaded automatically by the /thenn.new and /thenn.update commands.
version: 0.3.0
---

# thenn-builder

This skill defines the authoring protocol for `.thenn` workflow files. Follow these instructions when creating or updating a flow.

---

## .thenn file format

`.thenn` files are YAML with this top-level structure:

```yaml
name: flow-name           # required
description: "..."        # shown in /thenn.list
input: "Prompt text"      # optional: ask this if no args given at run time

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
prompt: "..."             # standalone: full inline prompt; combined with command/agent: appended as context
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
max_iterations: 5
until:
  type: bash
  run: grep -q "^status: pass" docs/review.md
steps:
  - id: review
    type: agent
    command: review-code
```

### When to use `type: coordinator`

The default behaviour for AI-driven steps is `type: agent` — spawn a fresh subagent. That is the right choice for the **majority** of steps: a subagent has a clean context and a focused task, which is exactly the "context-light" model thenn is built around.

Use `type: coordinator` only when the step genuinely cannot work as a subagent:

- **Agent-spawning** — the step needs to dispatch multiple subagents in parallel (a coordinator). A subagent cannot spawn further subagents, so this is impossible without `type: coordinator`.
- **Dynamic fan-out** — the step's work depends on which other steps have run, or on the flow's overall state, in a way a subagent with no view of the flow could not handle.
- **Mid-flow user interaction** — the step needs to call `AskUserQuestion` to clarify with the user, then continue based on the answer. (A subagent can do this too, but `type: coordinator` is the explicit form when you want the runner — not a black-box subagent — to handle it.)
- **Single complex action** — there is one heavy action (read three files, dispatch one investigation) that is more cheaply done in the main conversation than by spinning up a subagent.

For everything else, leave it as `type: agent`. The default keeps each step inspectable, replayable, and editable mid-flow.

### Coordinator voice for `type: coordinator` steps

When you use `type: coordinator`, the prompt you write is given to the **runner** (the main conversation), not to a subagent. The runner is the orchestrator; the convention is for the runner to be a **context-light coordinator** for this step.

Write the prompt in coordinator voice. The prompt should:

- **Be scoped** to this step's work. Do not ask the runner to do work that belongs to a later step.
- **Tell the runner what to dispatch, not what to do.** "Read X, then spawn a subagent to do Y" beats "Read X, do Y, then continue with Z." The runner should delegate the heavy work.
- **Specify outputs** — "write to .thenn-state/context.md" — so the next step can read them from disk.
- **Ask for a brief summary** — "return a one-line summary when done" — so the runner can hand control back to the flow.

**Do:**
- "Read .thenn-state/context.md. Spawn three subagents (one per angle) using the Agent tool. Each subagent writes findings to .thenn-state/findings-<angle>.md. Return a one-line summary per angle when done."
- "Use AskUserQuestion to ask the user which approach to take. Then write their choice to .thenn-state/chosen.md."

**Don't:**
- "Thoroughly analyse every file in the repo, implement fixes, run tests, and commit." (Too much for one step — split the work across the flow.)
- "Continue past this step and run the next one." (The runner is the orchestrator and will move to the next step after this one.)

---

## Deciding `input: true` vs `input: false`

Every `type: agent` and `type: coordinator` step must declare `input: true` or `input: false`. Apply this rule to every such step:

Ask: **"Could this step do its job correctly using only files on disk, without knowing what the user originally asked for?"**

- If **no** → `input: true`
- If **yes** → `input: false`

Common pattern: early steps that establish context (reproduce, plan, specify) often need `input: true` even if a previous step already ran, because files written so far may not capture the full human intent. Later steps that purely execute against an already-written plan or spec can use `input: false`. For `type: coordinator` steps, the runner reads `.thenn-state/input.md` itself and incorporates it into the work — so the same `input: true` / `input: false` choice applies, but the runner (not a subagent) does the reading.

---

## Generic scaffold (no description provided)

```yaml
name: <name>
description: Describe what this flow does

steps:
  # Run a slash command in a fresh context
  - id: step-one
    type: agent
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

---

## Scaffold protocol

Given a `<name>` and optional `<description>`:

1. Run `mkdir -p .thenn` to ensure the directory exists.
2. Check if `.thenn/<name>.thenn` already exists. If it does, use AskUserQuestion to confirm: "`.thenn/<name>.thenn` already exists. Overwrite?" with options `["Yes, overwrite", "Cancel"]`. If cancelled, stop.
3. If a description was provided, generate a complete, meaningful flow tailored to that workflow. Apply the `input: true / false` rule to every `type: agent` and `type: coordinator` step. Use `type: coordinator` for any step that genuinely needs to spawn its own subagents, fan out dynamically, or interact with the user mid-flow (see "When to use `type: coordinator`" above). Do not use the generic scaffold.
4. If no description was provided, use the generic scaffold above, substituting `<name>`.
5. Write the flow to `.thenn/<name>.thenn`.
6. Confirm to the user:
   ```
   Created .thenn/<name>.thenn
   Edit it to define your steps, then run it with: /thenn.run <name>
   ```

---

## Update protocol

Given a resolved `<file-path>` and optional `<change>` description:

1. Read the existing flow file with the Read tool.
2. If a change description was provided, apply it to the flow:
   - Add, remove, or reorder steps as described
   - Update `description`, `input`, or step fields as needed
   - Re-apply the `input: true / false` rule to any affected `type: agent` or `type: coordinator` steps
   - Switch a step between `type: agent` and `type: coordinator` where appropriate (see "When to use `type: coordinator`" above)
   - Preserve everything not mentioned in the change description
3. If no change description was provided, display the current flow to the user and use AskUserQuestion to ask: "What would you like to change?" with option `["Cancel"]`. Use the user's typed answer as the change description, then apply it per step 2. If the user selects Cancel, stop.
4. Write the updated content back to `<file-path>`.
5. Confirm to the user:
   ```
   Updated <file-path>
   Run it with: /thenn.run <name>
   ```
