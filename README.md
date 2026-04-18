<div align="center">
  <img src="artifacts/thenn-logo.svg" alt="thenn" />
</div>

> Simple, repeatable AI workflows. Define named flows that chain AI commands, agents, and bash steps — then just run them.

## Usage

```
/thenn run <name> [description]   Run a named flow
/thenn new <name> [description]   Scaffold a new flow
/thenn list                       List available flows
```

## How it works

Flows are `.thenn` files — simple YAML that sequences commands, agents, bash steps, and human gates into a named, repeatable workflow.

```yaml
name: bugfix
description: Reproduce, plan, implement, lint, review, commit
input: "Describe the bug"

steps:
  - id: reproduce
    type: claude
    input: true
    prompt: "Write a failing test that reproduces the bug. Do not fix it yet."

  - id: plan
    type: claude
    input: true
    prompt: "Read the failing test. Plan the fix and save it to spec/<branch>/plan.md"

  - id: review-plan
    type: human
    message: "Review spec/<branch>/plan.md, then press Continue"

  - id: implement
    type: claude
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
/thenn run bugfix the login button throws a TypeError on Safari
```

Scaffold a new flow from a description:

```
/thenn new bugfix reproduce bug -> plan fix -> human review -> implement -> lint -> commit
```

The thenn file is the wiring. The intelligence lives in your commands, agents, and prompts.

## Step types

| Type | What it does |
|---|---|
| `type: claude` | Runs a slash command, named agent, or inline prompt in a fresh subagent context |
| `type: bash` | Runs a shell command. `on_failure: stop \| continue \| human` |
| `type: human` | Pauses and waits for the user to press Continue |
| `type: loop` | Repeats sub-steps until a bash exit condition is met, up to `max_iterations` |

### `type: claude` options

```yaml
- id: specify
  type: claude
  input: true              # inject user's run-time description into this step
  command: speckit.specify # call a slash command (colon for subdirs: gsd:plan-phase)
  # agent: implementer     # or spawn a named agent
  # prompt: "..."          # or use an inline prompt
  # prompt: "..."          # combined with command/agent: appended as additional context
```

`input: true` or `input: false` is required on every `type: claude` step. Set `input: true` on any step that needs the user's original description — not just the first step.

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
      type: claude
      input: false
      prompt: "Review the code. Write docs/review.md: 'status: pass' or 'status: fail' + issues."
    - id: fix
      type: claude
      input: false
      prompt: "Read docs/review.md. Fix every issue listed."
```

## Installation

Copy or symlink the command and skill into your global Claude config:

```bash
# Clone
git clone https://github.com/Joe-Withers/thenn.git ~/thenn

# Symlink globally
mkdir -p ~/.claude/commands ~/.claude/skills/thenn-runner
ln -s ~/thenn/commands/thenn.md ~/.claude/commands/thenn.md
ln -s ~/thenn/skills/thenn-runner/SKILL.md ~/.claude/skills/thenn-runner/SKILL.md
```

Then in any project, create a `.thenn/` directory and add your flow files.

## Example flows

Five example flows are bundled in `flows/` as starting points. Copy any into your project's `.thenn/` directory and adapt:

| Flow | What it does |
|---|---|
| `example-make-feature` | Implement → review-fix loop → commit |
| `example-review-loop` | Implement → fix until review passes (max 4 iterations) → commit |
| `example-bug-fix` | Reproduce → diagnose → human gate → fix → verify → commit |
| `example-spec-kit` | Specify → clarify → plan → tasks → commit *(requires github/spec-kit)* |
| `example-gsd` | Spec → plan → review → execute *(requires gsd-build/get-shit-done)* |

## Core principles

**The filesystem is the context bus.** Steps communicate by writing and reading files. No in-memory variable passing between AI instances. Each step gets a fresh context and reads what it needs from disk — making steps inspectable, replayable, and editable mid-flow.

**Flows are wiring, not prompting.** A thenn node calls a command or agent — it doesn't define what that command does. Use `command:` to call existing slash commands; use `prompt:` only for simple one-liners or additional context.

**One file per flow.** Everything about a flow lives in a single `.thenn` file. No separate command registry, no split config.

## Related work

| Project | Relation |
|---|---|
| GSD / GSD v2 | Full spec-driven dev framework; thenn is a lighter orchestration layer |
| github/spec-kit | Spec-driven toolkit; thenn flows can call speckit commands |
| Archon | Full application (web UI, DB, CLI); much heavier |
| Just | CLI task runner (inspiration for simplicity); thenn is the AI-native equivalent |
| Claude Code skills/commands | thenn *calls* these — it's the layer above them |
