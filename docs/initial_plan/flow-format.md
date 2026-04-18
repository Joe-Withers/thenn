# Thenn file format

Thenn files live in `.thenn/` (project-level) or `~/.thenn/` (personal/global).

File extension: `.thenn` (YAML syntax)

## Structure

```yaml
name: spec-kit
description: Brief → clarification → spec → tasks → commit

steps:
  - id: clarify
    type: claude
    command: clarify-brief

  - id: spec
    type: claude
    command: write-spec

  - id: review-spec
    type: human
    message: "Review docs/spec.md. Edit if needed, then press enter to continue"

  - id: tasks
    type: claude
    command: generate-tasks

  - id: commit
    type: bash
    run: git add docs/ && git commit -m "chore: add spec and tasks"
    on_failure: stop
```

## Step types

### `type: claude`

Calls a Claude command or agent in a fresh context.

```yaml
- id: write-spec
  type: claude
  command: write-spec          # calls /write-spec slash command
```

```yaml
- id: implement
  type: claude
  agent: implementer           # spins up a named subagent
```

Inline prompt (escape hatch for simple one-liners — use sparingly):

```yaml
- id: summarise
  type: claude
  prompt: "Summarise docs/spec.md in one paragraph and write to docs/summary.md"
```

### `type: ai` (future)

For non-Claude AI agents — same interface, different runner.

```yaml
- id: write-spec
  type: ai
  agent: codex
  command: write-spec
```

### `type: bash`

Runs a shell command.

```yaml
- id: lint
  type: bash
  run: npm run lint
  on_failure: stop             # stop | continue | human (default: stop)
```

### `type: human`

Pauses the flow for human input or review.

```yaml
- id: review
  type: human
  message: "Review docs/spec.md. Edit if needed, then press enter to continue"
```

## Context passing

Steps share context by writing and reading documents on disk. There is no in-memory variable passing between steps.

Convention: commands/agents write their output to a predictable location (e.g. `docs/spec.md`) and subsequent commands are written to read from that location.

This means:
- Steps are fully inspectable mid-flow
- You can edit any document and re-run from that step
- Each AI instance gets a clean context window
- No risk of context rot across long flows

## `on_failure` options (bash steps)

| Value | Behaviour |
|---|---|
| `stop` | Abort the flow, report the error (default) |
| `continue` | Log the failure and proceed to the next step |
| `human` | Pause and ask the human what to do |

## Discovery

Thenn discovers flow files from:

1. `.thenn/` in the current project (project-level)
2. `~/.thenn/` (personal, available in all projects)

Project flows take precedence over personal flows with the same name.

## Planned: flow composition

Future support for calling one flow from within another:

```yaml
- id: run-spec-kit
  type: flow
  flow: spec-kit
```
