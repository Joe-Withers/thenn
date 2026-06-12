---
description: Resume a journaled .thenn flow, or start a new journaled run
argument-hint: "[ts]"
allowed-tools: Bash, Read, Glob, Write, Skill, Agent, AskUserQuestion
---

You are handling the `/thenn.resume` command. Arguments: `$ARGUMENTS`

This is the journaled counterpart to `/thenn.run`. Every step execution is recorded in `.thenn-state/runs/<ts>/`, so if the flow dies mid-run, invoking `/thenn.resume` again picks up at the next step after the last completed one. `/thenn.run` is unchanged — use it for fire-and-forget runs that don't need resumability.

## Step 1 — Parse arguments

The optional argument is a run timestamp `<ts>` (e.g. `20260612T193045Z`). If `$ARGUMENTS` is empty, `<ts>` is null and the command will look for any in-progress run.

If multiple tokens are provided, take the first as `<ts>` and ignore the rest (with a one-line note).

## Step 2 — Find or create the journal

Scan `.thenn-state/runs/*/run.yaml` for entries with `status: in_progress`. Use Glob + Read, or a single Bash command:

```bash
for d in .thenn-state/runs/*/run.yaml; do
  [ -f "$d" ] || continue
  status=$(grep -E "^status:" "$d" | awk '{print $2}')
  if [ "$status" = "in_progress" ]; then
    echo "$d"
  fi
done
```

| Situation | Action |
|---|---|
| `<ts>` given | Open `.thenn-state/runs/<ts>/run.yaml`. Verify `status: in_progress`. If the file is missing, error: "No journal at `.thenn-state/runs/<ts>/`." If `status` is anything other than `in_progress` (e.g. `completed`, `failed`, `stopped`), error: "Run `<ts>` is `<status>`, not `in_progress`. Nothing to resume. Use `/thenn.run <name>` to start a new run, or `/thenn.resume` with no args to look for another in-progress run." Then list available in-progress runs (the same scan as below). |
| `<ts>` omitted, exactly 1 in-progress run | AskUserQuestion: "Resume run `<ts>` for flow `<flow_name>` (resuming from step `<resume_from+1>`)? [Resume / Start fresh on a new flow / Cancel]". On Resume, proceed with that run. On Start fresh, set the chosen run aside (see below) and create a new one. |
| `<ts>` omitted, multiple in-progress runs | AskUserQuestion with one option per in-progress run (labeled `<flow_name>` — step `<resume_from+1>/<N>` — `<ts>`), plus "Start fresh on a new flow" and "Cancel". |
| `<ts>` omitted, zero in-progress runs | AskUserQuestion: "No in-progress runs found. Start a new journaled run? [yes / no]". On no, exit. On yes, ask the user which flow to start (use `/thenn.list` to enumerate, or accept a free-text name). If the flow cannot be resolved, error. |

**"Start fresh on a new flow" from any branch:**

1. If an old journal was chosen (Resume-from-list → Start fresh), rename `.thenn-state/runs/<ts>/` to `.thenn-state/runs/<ts>-superseded/` (or `<ts>-superseded-N/` on collision).
2. Generate a new `<ts>`:

   ```bash
   ts=$(date -u +%Y%m%dT%H%M%SZ)
   while [ -d ".thenn-state/runs/$ts" ]; do
     ts="${ts%Z}-$(( ${ts##*-} + 1 ))Z"
     # If no -N suffix yet, start at -2
   done
   # Robust version: use a simple counter loop
   ts=$(date -u +%Y%m%dT%H%M%SZ)
   if [ -d ".thenn-state/runs/$ts" ]; then
     i=2
     while [ -d ".thenn-state/runs/${ts%-*}-$i" ] && [ ! -d ".thenn-state/runs/${ts}-$i" ]; do
       # handle both with-suffix and no-suffix cases
       i=$((i+1))
     done
     # simpler: just append -N
     i=2
     while [ -d ".thenn-state/runs/${ts}-${i}" ]; do i=$((i+1)); done
     ts="${ts}-${i}"
   fi
   ```

3. `mkdir -p .thenn-state/runs/<ts>/steps` and create a placeholder `run.yaml`:

   ```yaml
   run_id: <ts>
   flow_name: <unknown yet>
   flow_path: <unknown yet>
   flow_hash: null
   input_path: <unknown yet>
   started_at: <now>
   finished_at: null
   status: in_progress
   resume_from: 0
   ```

   `flow_name`, `flow_path`, and `input_path` are filled in by Steps 3 and 4.

Track the chosen `<ts>` for subsequent steps. Call it `RESUMED_TS` in your reasoning.

## Step 3 — Resolve the flow file

**If resuming (existing journal):** read `flow_path` from `run.yaml`. Use the Read tool to load it. The flow file is already on disk; do not re-resolve from the name.

**If starting fresh:** follow the same discovery protocol as `/thenn.run` (see `commands/thenn.run.md:24-34`):

1. `.thenn/<name>.thenn`
2. `~/.thenn/<name>.thenn`

If not found in either location, output:
```
Flow '<name>' not found.
```
Then run the list logic (see `/thenn.list`) to show what flows are available, and stop. Clean up the placeholder `run.yaml` (delete `.thenn-state/runs/<ts>/`).

Once the flow file is resolved, update `run.yaml` with `flow_name` and `flow_path`. Compute `flow_hash` with `sha256sum <flow_path> | awk '{print $1}'` and write it to `flow_hash`.

## Step 4 — Capture or verify input

**If starting fresh:** follow `commands/thenn.run.md:36-46` to capture input. That is:

| Situation | Action |
|---|---|
| Trailing args provided (e.g. `add dark mode`) | Use the args as the input |
| No args, flow has `input` field | Use AskUserQuestion with the `input` value as the question, single option `["Continue"]`; capture what the user types as the input |
| No args, no `input` field | No input — skip |

If input was captured, run `mkdir -p .thenn-state` and write it to `.thenn-state/input.md`. Update `run.yaml.input_path` to `.thenn-state/input.md`.

**If resuming:** read `input_path` from `run.yaml`. Verify the file exists. If missing, error: "Journal `<ts>` references input at `<input_path>`, but the file is gone. To start a new journaled run, run `/thenn.resume` with no args and choose 'Start fresh'." If present, read it (it will be injected into steps per the SKILL protocol). If the file's content has been edited, the edited version is used — this is the documented way to change the input mid-run (see `docs/v0.3/journal.md`).

## Step 5 — Load the thenn-runner skill

Use the Skill tool to load `thenn-runner`. In your first message after loading, give the runner the resume context:

> Resume journal `<RESUMED_TS>` at `.thenn-state/runs/<RESUMED_TS>/`. The flow file is `<flow_path>`. Input is at `<input_path>`. `resume_from` is `<N>` in `run.yaml`. Follow the "Run journal" pre-flight in SKILL.md.

If starting fresh, say so: "Starting a fresh journaled run for flow `<flow_name>`. The journal is at `.thenn-state/runs/<RESUMED_TS>/`. `resume_from` is `0`. Follow the 'Run journal' pre-flight in SKILL.md."

## Step 6 — Execute

The runner follows the SKILL.md protocol, including the new "Run journal" pre-flight. The pre-flight reads `run.yaml` and starts at `resume_from`. Do not re-explain the step protocol — the skill contains it.
