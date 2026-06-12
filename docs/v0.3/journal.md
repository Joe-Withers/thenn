# Run journal + resume

Long `.thenn` flows die mid-run. Network drops, Ctrl-C, API timeouts, context exhaustion on a long `type: coordinator` step. Before the journal existed, the only way to recover was to re-invoke the flow from step 1 — and every step would re-run: re-spawn subagents, re-run `git commit` (which would then fail or double-commit), re-ask the user at human gates. Wasted time, lost work, and the only "skip if already done" logic in the entire runner was the `loop` step's pre-flight `until` check.

The **journal** is the durable version of the manual "edit `.thenn-state/*.md` then re-run" workflow. It records step start/end/status to disk under `.thenn-state/runs/<ts>/`. The new `/thenn.resume` command consults the journal and skips steps that have already completed, picking up at the next pending step.

`/thenn.run` is unchanged — use it for fire-and-forget flows that don't need resumability. Use `/thenn.resume` when you want a journaled run that can be picked up after a crash.

---

## How it works

When you invoke `/thenn.resume` (with or without a timestamp), the command either resumes an existing in-progress journal or starts a new journaled run for a flow. Every step the runner executes is recorded to disk as it happens. If the flow dies mid-run — Ctrl-C, network error, session timeout — the journal stays intact. The next `/thenn.resume` reads it, skips the steps that completed successfully, and continues.

A journaled run looks like this in `.thenn-state/`:

```
.thenn-state/
├── input.md                        # the user's input for this run
└── runs/
    └── 20260612T193045Z/           # <ts> = UTC, YYYYMMDDTHHMMSSZ
        ├── run.yaml                # run-level metadata
        └── steps/
            ├── 00.yaml             # one file per step, keyed on 0-based index
            ├── 01.yaml
            └── 02.yaml
```

`<ts>` is a timestamp in `YYYYMMDDTHHMMSSZ` format (UTC, ISO 8601 compact). On collision (two runs started in the same second), `-<n>` is appended. Step files are named `NN.yaml` (zero-padded 2 digits, then 3 once you pass 99) — the index is the stable key, not the `id` (the user may rename ids between runs).

### `run.yaml`

```yaml
run_id: 20260612T193045Z            # matches the parent directory name
flow_name: bugfix                   # the flow's `name:` field
flow_path: .thenn/bugfix.thenn      # resolved path used by /thenn.resume
flow_hash: 9f2a...                  # sha256 of the flow file at journal creation
input_path: .thenn-state/input.md   # the input.md used by this run
started_at: 2026-06-12T19:30:45Z
finished_at: null                   # set when the run ends (any terminal status)
status: in_progress                 # in_progress | completed | failed | stopped
resume_from: 0                      # 0-based index; updated as steps complete
```

`status` transitions: `in_progress` → `completed` (success) | `failed` (a step errored) | `stopped` (user aborted / `on_failure: stop` halted). `finished_at` is set on any terminal status.

### `steps/NN.yaml`

```yaml
index: 2
id: plan
type: agent
status: completed                   # pending | running | completed | failed | skipped
started_at: 2026-06-12T19:33:12Z
finished_at: 2026-06-12T19:35:00Z
result_summary: "Wrote docs/plan.md"
inputs_read:                        # informational
  - .thenn-state/input.md
  - docs/bug-analysis.md
outputs_written:                    # the key field for resume verification
  - docs/plan.md
```

`outputs_written` is the linchpin of the skip policy. For `type: agent` and `type: coordinator` steps, the runner refuses to mark the step `completed` without it. For `type: bash`, the field is set to `[]` (the step may or may not write files).

---

## `/thenn.resume` command

```
/thenn.resume                 # find the most recent in-progress run, ask to confirm
/thenn.resume 20260612T193045Z # resume that specific in-progress run
```

### With a timestamp

If `<ts>` is given, the command opens `.thenn-state/runs/<ts>/run.yaml` and verifies `status: in_progress`. If the file is missing, you get a clear error. If the run is `completed` / `failed` / `stopped`, you get a clear error and a list of available in-progress runs.

### Without a timestamp

The command scans for in-progress runs:

| In-progress runs found | Behavior |
|---|---|
| Exactly 1 | AskUserQuestion: "Resume run `<ts>` for flow `<flow_name>` (resuming from step `<N>`)? [Resume / Start fresh / Cancel]" |
| Multiple | AskUserQuestion with one option per run, plus "Start fresh on a new flow" and "Cancel" |
| Zero | AskUserQuestion: "No in-progress runs. Start a new journaled run? [yes / no]". On yes, ask which flow (use `/thenn.list` to enumerate). |

### "Start fresh on a new flow"

When you choose to start a new journaled run from a "what to resume?" prompt, the old run is moved aside:

```
.thenn-state/runs/20260612T193045Z/        # new run
.thenn-state/runs/20260612T193045Z-superseded/  # old run, kept for audit
```

The superseded run is kept indefinitely. You can `rm -rf` it manually if you want.

### Once the journal is set

The command resolves the flow file, captures or verifies `input.md`, and loads the `thenn-runner` skill. The skill's "Run journal" pre-flight reads `run.yaml` and starts at `resume_from`. From there, execution looks like a normal run, except the completed steps are skipped.

---

## Skip policy

Each step type has its own rules for "is this step already done?"

| Type | How "already done" is detected | Action on resume |
|---|---|---|
| `agent` | `status: completed` AND every path in `outputs_written` still exists on disk | Skip with `[thenn] Step X/N: [id] already completed, skipping`. If any output is missing, downgrade to `pending` and re-run. |
| `coordinator` | Same as `agent` | Same. |
| `bash` | `status: completed` (trust the journal) | Skip with the same announcement. **Re-running a non-idempotent command like `git commit` is exactly the failure mode this feature prevents.** |
| `human` | Always treated as `pending` on resume, regardless of journal | Re-ask the user. A human gate is a *confirmation* of current state, not a memo. |
| `loop` | The loop's `until` exit status | If `until` exits 0, skip the whole loop. Otherwise re-enter from iteration 0. |

The `outputs_written` check is the consistency guarantee. If the user deleted a file between runs (e.g. cleaned up `docs/`), the journal is downgraded and the step re-runs. This is cheap (one `ls` per file) and handles the "user cleaned up between runs" case.

For `type: bash` steps that are *not* idempotent — `git commit`, `npm publish`, anything with `append` semantics — the journal says "trust me" and skips. If the original run succeeded, the second invocation skipping is the right behavior. The flow author is responsible for making bash steps safe to skip (e.g. by checking state in the command, or by accepting that on resume after a crash, the side effect may have happened twice).

---

## Recovery semantics

What happens when a flow dies mid-step? The journal entry for that step has `status: running` and no `finished_at`. On resume, the runner:

1. Reads the journal.
2. For every step with `status: running` (regardless of type), treats it as `pending` — overwrites the YAML with `status: pending` and `finished_at: null` — and re-runs the step.

This applies uniformly to all step types:

- `bash` steps are typically idempotent (`npm test`, `ruff check`, `git status`) so re-running is safe. If the step has `on_failure: human`, the user is re-asked.
- `human` steps are re-asked (which is the desired behavior anyway).
- `coordinator` / `agent` steps re-evaluate from scratch against their `outputs_written` check.

The alternative — preserving in-flight prompt state — is too much ceremony for a rare case. The "treat `running` as `pending`" rule is safe in practice and easy to reason about.

---

## Editing the flow between runs

`flow_hash` is recorded in `run.yaml` at journal creation, but **it is not enforced in v1**. If you edit the flow file between runs — adding, removing, or renaming steps — `/thenn.resume` will still work. The index-based filename (`00.yaml`, `01.yaml`, ...) is the safety net: the runner keys off indices, not ids.

If `flow_hash` has changed, the runner prints a warning at the start of resume:

```
[thenn] Warning: flow file has changed since this run started (was <old>, now <new>).
Resuming at step <current_index+1> based on journal entries; verify the step still applies.
```

You can:
- Continue resume anyway (the journal is the source of truth for what has been done)
- Cancel and start a new journaled run (`/thenn.resume` with no args → "Start fresh")
- Manually edit `.thenn-state/runs/<ts>/steps/*.yaml` to force-re-run specific steps (set their `status: pending`)

---

## Lifecycle and cleanup

Journals are kept indefinitely. There is **no automatic garbage collection** in v1. Each run is small (a few KB of YAML), so the directory grows slowly.

To clean up:

```bash
# List all journals
ls -la .thenn-state/runs/

# Remove a specific run
rm -rf .thenn-state/runs/20260612T193045Z

# Remove all completed/failed/stopped runs (keep in_progress)
find .thenn-state/runs -name run.yaml -exec sh -c 'grep -q "status: in_progress" "$1" || rm -rf "$(dirname "$1")"' _ {} \;
```

The journal directory is already in `.gitignore` (or should be — see `docs/v0.1/input-args.md`); it is runtime state, not part of the repository.

---

## `/thenn.list` enhancement

`/thenn.list` now shows in-progress journaled runs at the top of its output:

```
In-progress journaled runs (use /thenn.resume to continue):
  bugfix        3/5 steps   2026-06-12T19:30:45Z
  feature       1/4 steps   2026-06-12T20:14:02Z

Project flows (.thenn/):
  bug-fix      Structured bug fix with reproduction and verification
  feature      Implement and review a feature

Run a flow: /thenn.run <name>
Resume a journaled run: /thenn.resume [ts]
```

This is a quick way to see "what did I have running?" without scanning the file system.

---

## Limitations (v1)

- **No captured step output.** The journal records *what was done* (status, `outputs_written`, summary), not the full transcript. Subagent outputs are not stored; the step's written files are.
- **No garbage collection.** Manual cleanup of `.thenn-state/runs/` is fine. Auto-GC is a v2 feature.
- **No `flow_hash` enforcement.** Recorded but ignored. A future version may refuse to resume on a stale flow.
- **Loops restart from iteration 0.** The journal records `last_iteration` but the runner currently re-enters the loop from the top. A future version may resume mid-loop.
- **Input is fixed at journal creation.** The user's `input.md` is captured when `/thenn.resume` starts the run. To change the input mid-resume, edit `.thenn-state/input.md` directly (the runner will read the edited version on the next step).
- **`/thenn.run` does not journal.** Use `/thenn.resume` for journaled runs. `/thenn.run` is unchanged at the command level.
- **No `on_failure: human` retry state.** The `on_failure: human` prompt re-fires on resume.

---

## Backwards compatibility

The journal is a new feature. Existing flows work unchanged. `/thenn.run` is unchanged. `/thenn.list` gains a new section (in-progress runs) but its existing behavior is preserved.

Flows that wrote their own state files to `.thenn-state/` continue to work — the new `runs/` subdirectory does not conflict with the existing `input.md` and similar files.
