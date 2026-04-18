<div align="center">
  <img src="artifacts/thenn-logo.svg" alt="thenn" />
</div>

> Simple, repeatable AI workflows. Define named flows that chain AI commands, agents, and bash steps — then just run them.

## What it is

A plugin for AI coding agents (Claude Code and beyond) that lets you define `.thenn` files: simple YAML files that sequence commands, agents, and bash steps into named, repeatable workflows.

```
/thenn run spec-kit
/thenn run e2e-review
/thenn list
```

The thenn file is the wiring diagram. The intelligence lives in your existing commands and agents.

## What it is not

- A prompt framework — prompts live in commands/agents, not in thenn files
- A context-passing system — steps share context via documents on disk, not in-memory variables
- A replacement for GSD, claude-orchestration, or Archon — it's intentionally simpler

## Core principles

**The filesystem is the context bus.** Steps communicate by writing and reading documents (`docs/spec.md`, `docs/tasks.md`). No magic variable piping between AI instances. Each step gets a fresh context and reads what it needs from disk. This makes steps inspectable, replayable, and editable mid-flow.

**Flows are wiring, not prompting.** A thenn node calls a command or agent — it doesn't define what that command does. 90% of nodes will be `command: write-spec` not `prompt: "write a spec that..."`.

**One file per flow.** Everything about a flow — its steps, human gates, bash steps — lives in a single `.thenn` file. No separate command registry, no split config.

**Spec-driven by default.** Flows are designed around the pattern of producing artifacts at each step (clarification notes, specs, task lists) rather than relying on an AI instance's context window accumulating state.

## Related work

| Project | Relation |
|---|---|
| GSD / GSD v2 | Full spec-driven dev framework; thenn is a lighter orchestration layer |
| mbruhler/claude-orchestration | Closest prior art; good execution primitives but no named flow registry, no bash-first-class, novel syntax |
| Archon | Full application (web UI, DB, CLI); much heavier |
| Just | CLI task runner (inspiration for simplicity); thenn is the AI-native equivalent |
| Claude Code skills/commands | thenn *calls* these — it's the layer above them |
