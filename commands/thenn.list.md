---
description: List available .thenn flows (project + personal) and any in-progress journaled runs
argument-hint: ""
allowed-tools: Bash, Read, Glob
---

You are handling the `/thenn.list` command.

## Step 1 — List in-progress journaled runs

Scan `.thenn-state/runs/*/run.yaml` for entries with `status: in_progress`.

For each, read `flow_name` and `resume_from`. To compute total steps, read the flow file at `flow_path` and count `steps:` entries (or just say `?` if the file is missing).

If any in-progress runs exist, output this section first:

```
In-progress journaled runs (use /thenn.resume to continue):
  bugfix        3/5 steps   2026-06-12T19:30:45Z
  feature       1/4 steps   2026-06-12T20:14:02Z
```

Format: two spaces between columns, padded so the timestamps align. Each line is `<flow_name>` + spaces + `<resume_from>/<total> steps` + spaces + `<ts>`.

If no in-progress runs exist (or `.thenn-state/runs/` doesn't exist), skip this section silently.

## Step 2 — List available flows

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
Resume a journaled run: /thenn.resume [ts]
Bundled starter flows: see the flows/ directory in the thenn plugin.
```

If a directory does not exist or is empty, show `(none)` for that section.

If both are empty, output:
```
No flows found. Create one with: /thenn.new <name>
Bundled starter flows are in the thenn plugin's flows/ directory — copy one to .thenn/ to get started.
```
