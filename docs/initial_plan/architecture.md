# Architecture

## Directory structure

```
thenn/
├── .claude-plugin/
│   └── plugin.json              # Plugin metadata
├── commands/
│   └── thenn.md                 # /thenn command (run, list, new)
├── skills/
│   └── thenn-runner/
│       └── SKILL.md             # Teaches the AI to interpret and execute .thenn files
├── flows/                       # Bundled starter flows
│   ├── spec-kit.thenn
│   ├── gsd.thenn
│   ├── e2e-review.thenn
│   └── bug-fix.thenn
└── README.md
```

## How it works

There is no external runtime. The plugin uses Claude Code's native primitives:

1. `/thenn run <name>` triggers the `thenn.md` command
2. The `thenn-runner` skill is loaded, teaching Claude the `.thenn` format
3. Claude locates the named flow file (project then personal)
4. Claude executes each step in sequence using its native tools:
   - `type: claude` → spawns a subagent or runs a slash command
   - `type: bash` → runs via Bash tool
   - `type: human` → pauses for human input
5. Each step completes before the next begins

No parser. No binary. No install beyond `/plugin install`.

## The `/thenn` command

```
/thenn run <name>     Run a named flow
/thenn list           List available flows (project + personal)
/thenn new <name>     Scaffold a new .thenn file
```

## The `thenn-runner` skill

A `SKILL.md` that:
- Describes the `.thenn` file format to the AI
- Defines how each step type is executed
- Handles `on_failure` logic for bash steps
- Knows to look in `.thenn/` then `~/.thenn/`

Because it's a skill (not a script), the AI can handle edge cases gracefully — pausing when ambiguous, asking for clarification on malformed steps, etc.

## Plugin metadata (`plugin.json`)

```json
{
  "name": "thenn",
  "version": "0.1.0",
  "description": "Simple, repeatable AI workflows. Define named flows that chain commands, agents, and bash steps.",
  "commands": ["thenn"],
  "skills": ["thenn-runner"]
}
```

## Installation (Claude Code)

```
/plugin install thenn
```

Or manually: copy `.claude-plugin/`, `commands/`, and `skills/` into your project's `.claude/` directory.

## AI agnostic design

Thenn is designed to work beyond Claude Code. The `.thenn` file format is AI-agnostic — `type: claude` is one runner, but `type: ai` with a configurable agent is the direction for broader support.
