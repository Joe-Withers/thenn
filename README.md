<div align="center">
  <img src="artifacts/thenn-logo.svg" alt="thenn" />
</div>

> Simple, repeatable AI workflows. Define named flows that chain AI commands, agents, and bash steps — then just run them.

## Installation

```
/plugin marketplace add Joe-Withers/thenn
/plugin install thenn@thenn
```

## Usage

```
/thenn.run <name> [input...]      Run a named flow
/thenn.new <name> [description]   Scaffold a new flow
/thenn.update <name> [change...]  Edit an existing flow
/thenn.list                       List available flows
```

## How it works

Flows are `.thenn` files — simple YAML that sequences commands, agents, bash steps, and human gates into a named, repeatable workflow.

```yaml
name: bugfix
description: Reproduce, plan, implement, lint, review, commit
input: "Describe the bug"

steps:
  - id: reproduce
    type: agent
    input: true
    prompt: "Write a failing test that reproduces the bug. Do not fix it yet."

  - id: plan
    type: agent
    input: true
    prompt: "Read the failing test. Plan the fix and save it to spec/<branch>/plan.md"

  - id: review-plan
    type: human
    message: "Review spec/<branch>/plan.md, then press Continue"

  - id: implement
    type: agent
    input: false
    prompt: "Implement the fix from spec/<branch>/plan.md. Run the test to confirm it passes."

  - id: lint
    type: bash
    run: ruff check . && ruff format --check .
    on_failure: human

  - id: commit
    type: bash
    run: git add -A && git commit -m "fix: resolve bug"
    on_failure: stop
```

Run it with:

```
/thenn.run bugfix the login button throws a TypeError on Safari
```

Scaffold a new flow from a description:

```
/thenn.new bugfix reproduce bug -> plan fix -> human review -> implement -> lint -> commit
```

The thenn file is the wiring. The intelligence lives in your commands, agents, and prompts.

## Step types

| Type | What it does |
|---|---|
| `type: agent` | Runs a slash command, named agent, or inline prompt in a fresh subagent context |
| `type: coordinator` | Same as `type: agent`, but the runner processes the step in the main conversation — required for steps that spawn subagents, fan out, or interact with the user mid-flow (see [Context-light coordinators](#context-light-coordinators) below) |
| `type: bash` | Runs a shell command. `on_failure: stop \| continue \| human` |
| `type: human` | Pauses and waits for the user to press Continue |
| `type: loop` | Repeats sub-steps until a bash exit condition is met, up to `max_iterations` |

### `type: agent` options

```yaml
- id: specify
  type: agent
  input: true              # inject user's run-time description into this step
  command: speckit.specify # call a slash command (colon for subdirs: gsd:plan-phase)
  # agent: implementer     # or spawn a named agent
  # prompt: "..."          # or use an inline prompt
  # prompt: "..."          # combined with command/agent: appended as additional context
```

`input: true` or `input: false` is required on every `type: agent` and `type: coordinator` step. Set `input: true` on any step that needs the user's original description — not just the first step.

### `type: loop`

```yaml
- id: review-fix-loop
  type: loop
  max_iterations: 3
  until:
    type: bash
    run: grep -q "^status: pass" docs/review.md
  steps:
    - id: review
      type: agent
      input: false
      prompt: "Review the code. Write docs/review.md: 'status: pass' or 'status: fail' + issues."
    - id: fix
      type: agent
      input: false
      prompt: "Read docs/review.md. Fix every issue listed."
```

## Example flows

Six example flows are bundled in `flows/` as starting points. Copy any into your project's `.thenn/` directory and adapt:

| Flow | What it does |
|---|---|
| `example-make-feature` | Implement → review-fix loop → commit |
| `example-review-loop` | Implement → fix until review passes (max 4 iterations) → commit |
| `example-bug-fix` | Reproduce → diagnose → human gate → fix → verify → commit |
| `example-coordinator` | Coordinator fan-out: gather context, three parallel investigations, synthesize, review |
| `example-spec-kit` | Specify → clarify → plan → tasks → commit *(requires github/spec-kit)* |
| `example-gsd` | Spec → plan → review → execute *(requires gsd-build/get-shit-done)* |

## Context-light coordinators

By default a `type: agent` step spawns a fresh subagent. That is the right model for the vast majority of steps. But three things are awkward for a subagent:

- **Agent-spawning** — a subagent cannot spawn further subagents.
- **Dynamic fan-out** — a subagent has no view of the flow outside its own prompt.
- **Mid-flow user interaction** — a subagent can call `AskUserQuestion`, but nothing in its prompt says "return to step N+1 when done," so a long step can make the orchestrator lose the thread.

For these cases, use `type: coordinator` instead. The runner (this conversation) processes the step in its own context instead of spawning a subagent. The runner can then use the Agent tool to dispatch sub-subagents, call `AskUserQuestion` to interact with the user, and make decisions based on flow state — none of which a subagent can do.

```yaml
- id: parallel-investigate
  type: coordinator        # runner does the work, can spawn subagents
  input: false
  prompt: |
    Read .thenn-state/context.md. Fan out to three parallel investigations
    using the Agent tool: performance, security, maintainability. Each
    subagent writes findings to .thenn-state/findings-<angle>.md. Do not
    redo the work yourself — your job is to coordinate. Return a
    one-line summary per angle when done. The thenn-runner will then
    move to the next step.
```

The convention is to write the step's `prompt:` in **coordinator voice** — be scoped, delegate heavy work, write outputs to disk, return a brief summary, then explicitly hand control back to the flow. The runner is the orchestrator; a long `type: coordinator` step must not make it forget which step it's on.

See `docs/v0.2/coordinator.md` for the full convention, including the return protocol and when to use `type: coordinator` vs `type: agent`. A bundled example lives at `flows/example-coordinator.thenn`.

## Core principles

**The filesystem is the context bus.** Steps communicate by writing and reading files. No in-memory variable passing between AI instances. Each step gets a fresh context and reads what it needs from disk — making steps inspectable, replayable, and editable mid-flow.

**Flows are wiring, not prompting.** A thenn node calls a command or agent — it doesn't define what that command does. Use `command:` to call existing slash commands; use `prompt:` only for simple one-liners or additional context.

**`type: coordinator` steps run in the main conversation.** When a step needs to spawn its own subagents, fan out, or interact with the user mid-flow, use `type: coordinator` and the runner processes it directly. The runner stays a context-light coordinator: it delegates the heavy work, writes outputs to disk, and hands control back to the flow. See [Context-light coordinators](#context-light-coordinators) above for details.

**One file per flow.** Everything about a flow lives in a single `.thenn` file. No separate command registry, no split config.

## Related work

| Project | Relation |
|---|---|
| GSD / GSD v2 | Full spec-driven dev framework; thenn is a lighter orchestration layer |
| github/spec-kit | Spec-driven toolkit; thenn flows can call speckit commands |
| Archon | Full application (web UI, DB, CLI); much heavier |
| Just | CLI task runner (inspiration for simplicity); thenn is the AI-native equivalent |
| Claude Code skills/commands | thenn *calls* these — it's the layer above them |
