# Flow input and argument passing

Flows can declare an `input` field that captures the user's intent before any steps run. This solves the "when do I describe the feature?" problem — the description travels with the run command and is available as context throughout the flow.

---

## Usage

Pass input as trailing arguments to `/thenn run`:

```
/thenn run spec-kit add a dark mode toggle to the settings page
/thenn run gsd migrate the auth module to use JWT
```

Everything after the flow name becomes the initial input for that run.

If no arguments are given and the flow declares an `input` field, thenn pauses and asks the question before any steps execute:

```
/thenn run spec-kit
→ [thenn] What do you want to build?
→ (user types answer, then presses Continue)
```

If no arguments are given and there is no `input` field, the flow runs without input — step commands handle their own input gathering as they normally would.

---

## Declaring input in a flow

```yaml
name: spec-kit
description: Spec-driven feature development — specify, clarify, plan, tasks, commit
input: "What do you want to build?"

steps:
  - ...
```

The `input` value is the question shown to the user when no args are provided.

---

## How input is stored and passed to steps

Before any steps run, thenn writes the captured input to `.thenn-state/input.md`:

```
add a dark mode toggle to the settings page
```

Steps opt in to receiving the input by setting `input: true`. Only steps that actually need the original description should set this — typically the first step. Later steps in the flow read from files written by earlier steps (spec.md, plan.md, etc.) and don't need the original description repeated.

```yaml
steps:
  - id: specify
    type: claude
    command: speckit.specify
    input: true          # inject user's request — speckit.specify needs to know what to spec

  - id: clarify
    type: claude
    command: speckit.clarify
                         # no input: true — reads from spec.md instead
```

When `input: true` is set and `.thenn-state/input.md` exists, the thenn-runner appends this block to the subagent prompt:

```
Run the /speckit.specify slash command and complete the task it describes.

User's request for this flow run:
add a dark mode toggle to the settings page
```

If `input: true` is set but no input was provided at run time, the block is simply omitted — no error.

`.thenn-state/` is runtime state and should be gitignored.

---

## Behaviour matrix

| Args passed | Flow has `input` field | Behaviour |
|---|---|---|
| Yes | Yes | Write args to `.thenn-state/input.md`, skip the question |
| Yes | No | Write args to `.thenn-state/input.md` |
| No | Yes | Ask the question, write answer to `.thenn-state/input.md` |
| No | No | No input file written; steps handle their own input gathering |

---

## Design notes

**Why a file and not in-memory passing?**  
Consistent with thenn's filesystem-as-context-bus principle. The input is inspectable, editable between steps, and replayable. If you want to tweak the description before the plan step runs, edit `.thenn-state/input.md` during a human gate.

**Why opt-in per step rather than injecting everywhere?**  
Most steps don't need the original description — they read from files written by earlier steps. `speckit.clarify` reads spec.md; `speckit.plan` reads spec.md; `speckit.tasks` reads plan.md. Injecting the original input into these steps is redundant at best and misleading at worst (the spec may have evolved from what the user originally typed). Steps that genuinely need it — typically only the first — declare `input: true`.

**Why not a `{input}` interpolation syntax in the flow YAML?**  
Interpolation in step fields (`command: speckit.specify --description "{input}"`) was considered and rejected for v0.1 — it's implicit, hard to inspect, and creates parsing complexity. Injection at the prompt level is simpler and more transparent.
