# `type: coordinator` — running a step in the orchestrator's conversation

By default, every AI-driven step in a `.thenn` flow uses `type: agent` and is delegated to a **fresh subagent**: the runner spawns a subagent via the Agent tool, the subagent does the step's work, and the subagent returns a result. That is the right model for the vast majority of steps — each subagent has a clean context and a focused task.

Some steps, however, cannot work as subagents. They need capabilities that only the **main conversation** has:

1. **Agent-spawning** — a subagent cannot spawn further subagents. A step that needs to dispatch work in parallel (a coordinator) must run in the main context, so the runner itself can call the Agent tool.
2. **Dynamic fan-out** — a subagent has no view of the flow outside its own prompt. A step whose work depends on which steps have run, or on the flow's overall state, must run in the main context.
3. **Mid-flow user interaction** — a step that needs to ask the user a question mid-step (e.g., "which approach?") and continue based on the answer benefits from running in the main context, so the orchestrator stays in charge of the thread.

For these cases, use `type: coordinator` on the step. The runner (this conversation) processes the step in its own context instead of spawning a subagent. The runner can use any tool it has access to, including the Agent tool to spawn sub-subagents and `AskUserQuestion` to interact with the user.

```yaml
- id: parallel-investigate
  type: coordinator        # runner does the work, can spawn subagents
  input: false
  prompt: |
    Read .thenn-state/context.md. Spawn three subagents in parallel using
    the Agent tool — performance, security, maintainability. Each writes
    findings to .thenn-state/findings-<angle>.md. Return a one-line
    summary per angle when done. The thenn-runner will then move to
    the next step.
```

`type: coordinator` accepts the same fields as `type: agent` (`command:`, `agent:`, `prompt:`, `input:`). The only difference is **who does the work** — for `type: agent`, a subagent; for `type: coordinator`, the runner.

---

## How it works

When the runner encounters a `type: coordinator` step:

1. It announces the step: `[thenn] Step X/N: [step-id] <brief action description>`.
2. It processes the step's task **in the main conversation**:
   - If the step has a `prompt:`, that is the runner's task.
   - If the step has a `command:`, the runner invokes the slash command in the main conversation (using the Skill tool if the command is a `SKILL.md`).
   - If the step has an `agent:`, the runner spawns that agent using the Agent tool — but the runner is in charge, so it can spawn multiple instances, none, or in a different order than the `agent` value alone would suggest.
   - Any `prompt` value combined with `command` or `agent` is treated as additional context that steers the work.
   - If `input: true`, the runner reads `.thenn-state/input.md` and incorporates the user's request.
3. The runner uses whatever tools it has access to — Bash, Read, Write, Edit, Agent, AskUserQuestion, etc. This is the whole point of `type: coordinator`: the runner can now do the things a subagent cannot.
4. When the step is done, the runner announces: `[thenn] Step X/N: [step-id] done`.
5. The runner moves to the next step in the flow YAML.

The step is still inspectable, replayable, and editable mid-flow, just like any other step. The user reads the runner's progress in the terminal — they see the step announced, the work happen, and the step complete.

---

## The context-light coordinator convention

A `type: coordinator` step's runner-side work is a **context-light coordinator** action. The runner's main context grows with each step (it processes each `type: coordinator` step itself), so a step that does too much will bloat the runner. The convention exists to keep steps small and the runner's context manageable.

**Do:**
- **Delegate heavy work to subagents** via the Agent tool. A coordinator's job is to dispatch, not to do.
- **Interact with the user via `AskUserQuestion`** for mid-step clarifications.
- **Write outputs to disk** so downstream steps can read them. Filesystem-as-context-bus still applies — the runner is not exempt from it.
- **Return a brief one-line summary** when the step is done.

**Don't:**
- **Do not try to execute future flow steps.** The runner is the orchestrator; it will move to step N+1 after this one. Do not pre-empt.
- **Do not modify the flow file or runner state.** Only write to the working directory.
- **Do not get lost in the step.** Always know "this is step X of N in flow Y; when done, move to X+1." A long step must not make the runner forget where it is in the flow.

### The return protocol

After the step's work is done, the runner must explicitly return to the flow:

1. Announce completion: `[thenn] Step X/N: [step-id] done`
2. Move on to the next step in the flow YAML, applying the normal execution rules.

This is what stops a long-running `type: coordinator` step from making the orchestrator "lose the thread." The runner is also the orchestrator — the convention keeps the two roles from getting muddled when a step takes a long time.

---

## When to use `type: coordinator` vs `type: agent`

Use `type: agent` (spawn subagent) when:
- The step is a single, self-contained piece of work (write a file, run a command, read-and-summarise).
- You do not need the subagent to spawn more subagents or to interact with the user mid-step.
- The convention is "context-light" — the subagent gets a fresh context and a focused task.

Use `type: coordinator` (runner does the work) when:
- The step needs to **fan out** to multiple subagents in parallel (a coordinator dispatching work).
- The step needs to **ask the user** something mid-step (e.g., a clarification that determines the rest of the flow).
- The step needs to **make a decision** based on which steps remain, or based on the overall flow state, in a way a subagent with a fresh context could not.
- The step is a **single complex action** that the runner can do more cheaply in the main conversation than by spawning a subagent.

In short: `type: agent` is for **work**; `type: coordinator` is for **coordination**.

---

## Authoring examples

### Simple fan-out

```yaml
- id: parallel-investigate
  type: coordinator
  input: false
  prompt: |
    Read .thenn-state/context.md. Spawn three subagents in parallel using
    the Agent tool — performance, security, maintainability. Each
    subagent writes findings to .thenn-state/findings-<angle>.md. Do not
    redo the work yourself — your job is to coordinate. Return a
    one-line summary per angle when done. The thenn-runner will then
    move to the next step.
```

The runner reads the context, spawns three subagents in parallel, waits for them, summarises, and moves on. Each subagent does its own scoped piece of work in its own context; the runner's context only grows by the size of the three summaries.

### Mid-flow user interaction

```yaml
- id: pick-approach
  type: coordinator
  input: false
  prompt: |
    Read .thenn-state/options.md (three implementation approaches with
    tradeoffs). Use AskUserQuestion to ask the user which approach to
    take. Then write their choice to .thenn-state/chosen.md and return
    a one-line confirmation. The thenn-runner will then move to the
    next step.
```

The runner asks the user, the user answers, the runner writes the result to disk and returns. The flow is not interrupted — the user just sees a question appear briefly while the coordinator runs.

### Anti-pattern: heavy doer

```yaml
# AVOID
- id: do-everything
  type: coordinator
  input: true
  prompt: |
    Read the user's request. Thoroughly analyse every file in the
    repository, then implement the fix across all affected files,
    run the test suite, fix anything that breaks, commit, and
    report back.
```

This tries to do the rest of the flow in one step. It is brittle (a long step with many failure modes), and it bloats the runner's context with the entire implementation. Split this into several steps in the flow YAML, or delegate the heavy work to a subagent with a clear brief.

---

## Interaction with other step types

`type: coordinator` is one of two AI-driven step types in thenn. It is distinct from:

- `type: agent` — the default; the runner spawns a subagent that does the work. Use this for ordinary, self-contained steps.
- `type: bash` — runs a shell command. No subagent, no main-context step.
- `type: human` — pauses and asks the user to continue.
- `type: loop` — runs the sub-steps using the same rules as the top level. A `type: coordinator` sub-step inside a loop works exactly the same way.

A `type: coordinator` step can use `AskUserQuestion` mid-step to interact with the user without leaving the flow. It can also write files for a later `type: human` step to review — the human gate still works the same way.

---

## Backwards compatibility

`type: coordinator` is a new step type. The pre-0.3 step type was `type: claude`; the pre-0.3 main-context option was `context: main`. Both are removed in v0.3.0 — existing flows must be updated to use `type: agent` (for what was `type: claude`) and `type: coordinator` (for what was `type: claude` with `context: main`).
