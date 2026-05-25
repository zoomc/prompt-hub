# Runtime Protocol Notes

## Lifecycle rules
- Worker sessions are disposable: one TASK, validate, report, exit.
- Session1 is the long-lived orchestrator; it owns spec/plan/checklist/reconcile and spawns the next worker only after verifying the previous result.
- Do not let a worker accumulate implementation context across multiple TASKs.
- If implementation starts to drift, stop the worker and rebuild from clean context.

## Context pollution signals
Treat a session as polluted when any of these appear:
- repeated failed fixes
- growing patch complexity
- unrelated file edits
- speculative rewrite
- architecture drift
- endless patch loops
- forgetting the original task
- random modifications

## Required response to pollution
- stop current worker
- discard the polluted session
- restart from the smallest clean spec/task boundary

## Execution hygiene
- Prefer minimal diff and reversible changes.
- Avoid future-proof abstraction, config explosion, and speculative refactors.
- Use small commits and localized edits.
- Keep prompt isolation strict: each prompt should expose only the current phase/TASK.
