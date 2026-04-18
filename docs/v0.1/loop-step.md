# Loop step type

Added in v0.1. Enables iterative flows — run a sequence of steps repeatedly until an exit condition is met.

The canonical use case is review → fix cycles: run a review agent, check whether it passed, fix if not, repeat.

---

## Format

```yaml
- id: review-fix-loop
  type: loop
  max_iterations: 5          # required — integer >= 1
  until:                     # required — bash check; exit code 0 = condition met
    type: bash
    run: grep -q "PASS" docs/review.md
  steps:                     # required — sub-steps to repeat
    - id: review
      type: claude
      command: review-code   # writes PASS or FAIL to docs/review.md

    - id: fix
      type: claude
      agent: implementer
```

### Fields

| Field | Required | Description |
|---|---|---|
| `id` | yes | Identifier for the loop step, used in progress output |
| `type` | yes | Must be `loop` |
| `max_iterations` | yes | Maximum number of times to run the sub-steps. Must be >= 1. |
| `until` | yes | Exit condition. A bash step whose exit code determines whether the loop is done (0 = done, non-zero = keep going). |
| `steps` | yes | The sub-steps to run each iteration. Supports `claude`, `bash`, and `human` step types. |

### `until` step

The `until` condition is always `type: bash`. It has the same fields as a regular bash step, except `on_failure` is not used — if the command errors, it is treated as "condition not yet met" and the loop continues.

```yaml
until:
  type: bash
  run: grep -q "PASS" docs/review.md
```

The command should:
- Exit 0 when the loop should stop (condition is met)
- Exit non-zero when the loop should continue

---

## Execution algorithm

```
1. Evaluate `until` condition
   └─ Exit 0 → skip the loop entirely ("condition already met")
   └─ Non-zero or error → proceed to step 2

2. Run each sub-step in sequence (iteration 1)
   └─ If a sub-step fails and stops → halt the entire flow

3. Evaluate `until` condition
   └─ Exit 0 → exit the loop ("converged after N iteration(s)")
   └─ Non-zero or error → increment iteration counter

4. If iteration counter >= max_iterations → stop the flow ("did not converge")

5. Go to step 2
```

The `until` condition is checked before the first iteration. This means if the condition is already satisfied when the flow reaches the loop step, the sub-steps are never run.

---

## Progress output

```
[thenn] Loop [review-fix-loop] — checking condition before iteration 1/5
[thenn] Loop [review-fix-loop] — condition not met, starting iteration 1/5
[thenn]   Step 1/2: [review] Running /review-code...
[thenn]   Step 1/2: [review] done
[thenn]   Step 2/2: [fix] Spawning implementer agent...
[thenn]   Step 2/2: [fix] done
[thenn] Loop [review-fix-loop] — iteration 1/5 complete, re-checking condition...
[thenn] Loop [review-fix-loop] — condition not met, starting iteration 2/5
[thenn]   Step 1/2: [review] Running /review-code...
[thenn]   Step 1/2: [review] done
[thenn] Loop [review-fix-loop] — iteration 2/5 complete, re-checking condition...
[thenn] Loop [review-fix-loop] — converged after 2 iteration(s)
```

Skipped (condition already met on entry):
```
[thenn] Loop [review-fix-loop] — condition already met, skipping
```

Max iterations reached:
```
[thenn] Loop [review-fix-loop] — did not converge after 5 iteration(s). Stopping flow.
```

Sub-step IDs are scoped to the loop in progress output: `[loop-id/step-id]` — this prevents confusion when the same sub-step id appears in multiple iterations.

---

## Error handling within loop sub-steps

Sub-steps follow the same error handling rules as top-level steps:

- `type: bash` sub-steps with `on_failure: stop` will halt the entire flow if they fail
- `type: bash` sub-steps with `on_failure: continue` will log and proceed to the next sub-step in that iteration
- `type: bash` sub-steps with `on_failure: human` will pause and ask, same as at the top level
- `type: claude` sub-steps that fail will halt the flow (no `on_failure` for claude steps)

If any sub-step halts the flow, the loop does not attempt further iterations.

---

## Design decisions

**`max_iterations` is required, not optional.** An uncapped loop in an AI-executed context is a footgun. Even if you expect the loop to converge quickly, the cost of an runaway loop (time, tokens) is high enough that an explicit limit is always the right call. Pick a generous number; it rarely triggers.

**`until` is evaluated before the first iteration.** This makes the loop behave like a `while` loop, not a `do-while`. If the condition is already passing when the flow reaches the loop, the sub-steps are skipped — which is almost always the right behavior.

**Exit condition is bash only.** The exit condition needs to be fast, deterministic, and cheap — a bash check against a file on disk fits perfectly. Using a `type: claude` exit condition would be expensive, slow, and non-deterministic. If you need Claude to make a qualitative judgment, have the review command write an explicit PASS/FAIL signal to a file, then check that file with bash.

**Context via filesystem.** The review command writes its output to disk (`docs/review.md`). The `until` condition reads from that same file. This is the same filesystem-as-context-bus principle as the rest of thenn — no in-memory state, everything is inspectable and editable between iterations.

---

## Example: review-fix loop

Full example for a code review → fix cycle:

```yaml
name: review-loop
description: Iterative code review — fix until review passes

steps:
  - id: initial-impl
    type: claude
    agent: implementer

  - id: review-fix-loop
    type: loop
    max_iterations: 4
    until:
      type: bash
      run: grep -q "^status: pass" docs/review.md
    steps:
      - id: review
        type: claude
        command: review-code        # writes docs/review.md with "status: pass" or "status: fail"

      - id: fix
        type: claude
        agent: implementer          # reads docs/review.md, applies fixes

  - id: commit
    type: bash
    run: git add -A && git commit -m "feat: implement with review"
    on_failure: stop
```

The `review-code` command is responsible for writing a consistent signal to `docs/review.md`. A simple convention:

```
status: pass
# All checks passed. Ready to commit.
```
or
```
status: fail
# Issues found: ...
```

The `until` condition checks for `status: pass` at the start of a line. The bash check is intentionally simple — no parsing, no ambiguity.

---

## Edge cases

| Situation | Behavior |
|---|---|
| `max_iterations` missing | Abort: "`loop` step requires `max_iterations`" |
| `until` missing | Abort: "`loop` step requires `until`" |
| `steps` missing or empty | Warn and skip the loop — treat as a no-op |
| `max_iterations: 0` | Treat as 1, warn: "max_iterations must be >= 1, using 1" |
| `until` command errors | Treat as "condition not met", continue looping |
| Sub-step fails with `on_failure: stop` | Halt the entire flow immediately |
| Condition met on first check | Skip all iterations ("condition already met") |
| Loops nested within loops | Supported — same execution rules apply recursively |
