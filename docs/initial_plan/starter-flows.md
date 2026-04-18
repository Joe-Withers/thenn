# Starter flows

Bundled flows that install with thenn. Copy and customise — they're meant as starting points.

---

## spec-kit

Brief → clarification → spec → human review → tasks → commit

```yaml
name: spec-kit
description: Spec-driven feature development from a brief

steps:
  - id: clarify
    type: claude
    command: clarify-brief

  - id: spec
    type: claude
    command: write-spec

  - id: review-spec
    type: human
    message: "Review docs/spec.md. Edit if needed, then press enter to continue"

  - id: tasks
    type: claude
    command: generate-tasks

  - id: commit-docs
    type: bash
    run: git add docs/ && git commit -m "chore: add spec and tasks"
    on_failure: stop
```

---

## gsd

Quick task → plan → first action. No heavy planning, no ceremony.

```yaml
name: gsd
description: Get something done — quick decomposition and first action

steps:
  - id: plan
    type: claude
    command: quick-plan

  - id: review-plan
    type: human
    message: "Review docs/plan.md. Adjust scope if needed, then press enter"

  - id: execute
    type: claude
    agent: implementer
```

---

## e2e-review

Full end-to-end with spec, implementation, and review gates.

```yaml
name: e2e-review
description: Full feature flow with human review at each major phase

steps:
  - id: clarify
    type: claude
    command: clarify-brief

  - id: spec
    type: claude
    command: write-spec

  - id: review-spec
    type: human
    message: "Review docs/spec.md before implementation begins"

  - id: implement
    type: claude
    agent: implementer

  - id: test
    type: bash
    run: npm test
    on_failure: human

  - id: code-review
    type: claude
    command: review-code

  - id: review-output
    type: human
    message: "Review implementation and code review notes. Approve or request changes"

  - id: commit
    type: bash
    run: git add -A && git commit -m "feat: implement feature per spec"
    on_failure: stop
```

---

## bug-fix

Reproduce → root cause → fix → verify → commit.

```yaml
name: bug-fix
description: Structured bug fix with reproduction and verification

steps:
  - id: reproduce
    type: claude
    command: reproduce-bug

  - id: root-cause
    type: claude
    command: find-root-cause

  - id: review-diagnosis
    type: human
    message: "Review docs/bug-analysis.md. Confirm root cause before fix"

  - id: fix
    type: claude
    agent: implementer

  - id: verify
    type: bash
    run: npm test
    on_failure: human

  - id: commit
    type: bash
    run: git add -A && git commit -m "fix: resolve bug per analysis"
    on_failure: stop
```

---

## Notes on commands referenced above

These flows assume the following commands exist in your project or personal Claude setup. They are not bundled — you write them to match your stack and conventions:

| Command | What it should do |
|---|---|
| `clarify-brief` | Read the brief (e.g. `docs/brief.md`), output `docs/clarification.md` |
| `write-spec` | Read brief + clarification, output `docs/spec.md` |
| `generate-tasks` | Read spec, output `docs/tasks.md` |
| `quick-plan` | Read a task description, output `docs/plan.md` |
| `review-code` | Review recent changes, output `docs/review-notes.md` |
| `reproduce-bug` | Reproduce a bug, output `docs/bug-analysis.md` |
| `find-root-cause` | Analyse bug, append root cause to `docs/bug-analysis.md` |

The `implementer` agent is a general-purpose coding agent — use Claude Code's built-in or define your own.
