---
name: thenn-builder
description: Use this skill when scaffolding or updating a .thenn workflow file. Teaches Claude how to generate and modify flows, apply the input injection rules, and write the file to .thenn/. Loaded automatically by the /thenn new and /thenn update commands.
version: 0.2.0
---

# thenn-builder

This skill defines the authoring protocol for `.thenn` workflow files. Follow these instructions when creating or updating a flow.

---

## .thenn file format

`.thenn` files are YAML with this top-level structure:

```yaml
name: flow-name           # required
description: "..."        # shown in /thenn list
input: "Prompt text"      # optional: ask this if no args given at run time

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
prompt: "..."             # standalone: full inline prompt; combined with command/agent: appended as context
input: true               # true: inject user's run-time description; false: read context from disk
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
    type: claude
    command: review-code
```

---

## Deciding `input: true` vs `input: false`

Every `type: claude` step must declare `input: true` or `input: false`. Apply this rule to every such step:

Ask: **"Could this step do its job correctly using only files on disk, without knowing what the user originally asked for?"**

- If **no** → `input: true`
- If **yes** → `input: false`

Common pattern: early steps that establish context (reproduce, plan, specify) often need `input: true` even if a previous step already ran, because files written so far may not capture the full human intent. Later steps that purely execute against an already-written plan or spec can use `input: false`.

---

## Generic scaffold (no description provided)

```yaml
name: <name>
description: Describe what this flow does

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

---

## Scaffold protocol

Given a `<name>` and optional `<description>`:

1. Run `mkdir -p .thenn` to ensure the directory exists.
2. Check if `.thenn/<name>.thenn` already exists. If it does, use AskUserQuestion to confirm: "`.thenn/<name>.thenn` already exists. Overwrite?" with options `["Yes, overwrite", "Cancel"]`. If cancelled, stop.
3. If a description was provided, generate a complete, meaningful flow tailored to that workflow. Apply the `input: true / false` rule to every `type: claude` step. Do not use the generic scaffold.
4. If no description was provided, use the generic scaffold above, substituting `<name>`.
5. Write the flow to `.thenn/<name>.thenn`.
6. Confirm to the user:
   ```
   Created .thenn/<name>.thenn
   Edit it to define your steps, then run it with: /thenn run <name>
   ```

---

## Update protocol

Given a resolved `<file-path>` and optional `<change>` description:

1. Read the existing flow file with the Read tool.
2. If a change description was provided, apply it to the flow:
   - Add, remove, or reorder steps as described
   - Update `description`, `input`, or step fields as needed
   - Re-apply the `input: true / false` rule to any affected `type: claude` steps
   - Preserve everything not mentioned in the change description
3. If no change description was provided, display the current flow to the user and use AskUserQuestion to ask: "What would you like to change?" with option `["Cancel"]`. Use the user's typed answer as the change description, then apply it per step 2. If the user selects Cancel, stop.
4. Write the updated content back to `<file-path>`.
5. Confirm to the user:
   ```
   Updated <file-path>
   Run it with: /thenn run <name>
   ```
