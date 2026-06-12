---
description: List available .thenn flows (project + personal)
argument-hint: ""
allowed-tools: Bash, Read, Glob
---

You are handling the `/thenn.list` command.

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

Run a flow: /thenn.run <name>
Bundled starter flows: see the flows/ directory in the thenn plugin.
```

If a directory does not exist or is empty, show `(none)` for that section.

If both are empty, output:
```
No flows found. Create one with: /thenn.new <name>
Bundled starter flows are in the thenn plugin's flows/ directory — copy one to .thenn/ to get started.
```
