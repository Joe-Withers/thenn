# Open questions and future work

Decisions deferred — don't block v0.1.

---

## Context passing (future)

Currently: steps communicate via documents on disk. This is intentional — it keeps each step independent, inspectable, and replayable.

Future: lightweight structured output passing for cases where writing a file is overkill (e.g. a bash step that returns a pass/fail that influences branching).

Options considered:
- `{varName}` interpolation (claude-orchestration style) — rejected for v0.1, too implicit
- Exit codes only for bash → branch logic — likely the right approach
- A `.thenn-state/` scratch directory per run for structured outputs

The principle holds: **don't rely on a Claude instance's context window as state**. Any passing mechanism should be explicit and inspectable.

---

## Branching / conditionals (future)

```yaml
- id: test
  type: bash
  run: npm test
  on_failure: human          # already planned

# Future:
- id: test
  type: bash
  run: npm test
  on_success: deploy
  on_failure: fix-tests
```

Keeping it sequential for v0.1. Conditionals add significant complexity to the skill.

---

## Flow composition (future)

Calling one flow from within another:

```yaml
- id: run-spec-kit
  type: flow
  flow: spec-kit
```

Useful for building larger flows from smaller ones. Deferred until the base format is stable.

---

## Parallelism (future)

Running steps concurrently (inspired by claude-orchestration's `[a || b || c]` syntax):

```yaml
- id: parallel-review
  type: parallel
  steps:
    - type: claude
      command: security-review
    - type: claude
      command: performance-review
```

Not needed for spec-driven sequential flows. Consider when use cases demand it.

---

## Resume / replay (future)

If a flow fails or is interrupted mid-run, resume from the failed step rather than restarting. Requires persisting run state (step completion, which files were written).

A `.thenn-state/<run-id>/` directory could track this. Low priority while flows are short.

---

## `on_failure: human` UX

When a bash step fails and `on_failure: human` is set, Claude should:
1. Show the error output
2. Ask what to do: retry / skip / abort / fix manually
3. Continue based on the answer

The exact interaction needs to be defined in the skill.

---

## Naming and discovery

Currently: `.claude/flows/` (project) and `~/.claude/flows/` (personal).

Questions:
- Should flows be namespaced by plugin? (e.g. `thenn:spec-kit`) — probably not, keep it flat
- Should thenn ship flows to `~/.claude/flows/` on install, or keep them in thenn directory and reference them? — reference from plugin dir, user can copy to customise
