# Spec Gate

## Purpose

Convert the user's informal requirement into a compact, implementation-ready specification before any coding work begins.

This skill is a gate, not a full project management process.

Its purpose is to prevent scope drift, premature implementation, unrelated edits, over-engineering, and treating uncertainty as confirmed fact.

## Aliases

- Spec Gate
- SG
- Requirement Gate
- Scope Gate
- Run Spec Gate
- Gate this first

## Core Rule

When this skill is triggered, do not start implementation.

Do not modify files.  
Do not run commands.  
Do not inspect code.  
Do not search the project.  
Do not check git status.  
Do not read files.  
Do not create patches.  
Do not start a coding plan.

The first action must be to rewrite the user's requirement into the fixed `Spec Gate` format.

Then stop and wait for explicit user confirmation.

Implementation may only start after the user clearly confirms, for example:

- confirm
- approved
- OK
- go
- start
- proceed
- do it
- implement it
- looks good
- use this

If the user does not confirm, stop.

## State Model

This skill has three simple states:

```text
idle -> spec_gate -> approved
```

### idle

No active gated task exists.

If the user asks for code work, project changes, debugging that requires project inspection, UI changes, business logic changes, configuration changes, testing, or implementation work, enter the `spec_gate` state.

### spec_gate

The requirement has been received but has not been approved by the user.

Only text response is allowed.

The agent must output the Spec Gate document and wait.

No tool calls are allowed in this state.

This includes “just checking”, “gathering context”, “quick inspection”, “reading relevant files”, “checking current behavior”, or “checking git status”. These are not exceptions.

If the current behavior is unknown, write `TBD`.

If the implementation depends on unknown project details, write them under `Assumptions` or `Open Questions`.

### approved

The user has explicitly confirmed the Spec Gate.

The agent may now inspect relevant files, plan internally, modify code, run tests, and report results.

## Mandatory Abort Rule

Before any tool call, check the current state.

If the current state is `spec_gate`, tool calls are forbidden.

If a tool call would happen in the `spec_gate` state, abort with this exact message:

```text
ABORT: tool call attempted before Spec Gate confirmation.
Allowed action: text-only Spec Gate output.
No further tool calls will be made this turn.
```

Do not retry.  
Do not use an alternative tool.  
Do not continue processing.  
The user must confirm or modify the Spec Gate first.

## Principle

Think before coding.

Clarify the target before touching the system.

Prefer a small, accurate spec over a large, fake-complete document.

Do not over-specify details that a modern coding agent can infer from the codebase after approval.

Do not invent APIs, file names, component names, schemas, commands, routes, test names, or implementation details unless the user explicitly mentioned them.

If something is unclear but not blocking, put it under `Assumptions`.

If something could materially change the implementation, put it under `Open Questions`.

## Output Format

Always output exactly this format:

```md
# Spec Gate

## Goal

Describe what should be achieved in 1-3 sentences.

## Scope

List what must be changed in this work.

## Out of Scope

List what must not be changed.

## Current Behavior

Describe the current behavior, problem, limitation, or known gap. Use `TBD` if unknown.

## Expected Behavior

Describe what should happen after the change.

## Business Rules

List business rules, edge cases, validation rules, permissions, state behavior, and must-not-break constraints.

## UI Requirements

Describe PC / Mobile / Responsive behavior, text, states, loading, empty state, error state, and interaction requirements.

## Assumptions

List inferred assumptions. Use `None` if there are no meaningful assumptions.

## Open Questions

List only blocking or high-impact questions. Use `None` if there are none.

## Gate Checklist

- Goal is clear.
- Scope is bounded.
- Out-of-scope is explicit.
- Current behavior is captured or marked as TBD.
- Expected behavior is explicit.
- Business rules are captured.
- UI requirements are captured.
- Assumptions are visible.
- Blocking questions are listed.
- No implementation has started.
```

End every Spec Gate response with:

```md
Please confirm whether this Spec Gate is correct. I will not start implementation until you confirm.
```

## Anti-Drift Rules

Protect the work from these common failures:

- Do not expand scope without permission.
- Do not refactor unrelated code.
- Do not rewrite nearby code just because it looks imperfect.
- Do not change unrelated UI copy, styles, routes, APIs, configuration, tests, or data.
- Do not add new abstractions unless required by the approved scope.
- Do not replace real behavior with mock behavior unless explicitly requested.
- Do not leave temporary bypass, debug, or mock logic in production paths.
- Do not treat assumptions as confirmed facts.
- Do not claim completion without verification.
- Do not start implementation before confirmation.

## Simplicity Rule

The best implementation is the smallest correct change that satisfies the approved spec.

Avoid cleverness.

Avoid broad rewrites.

Match the existing code style.

Every changed line should be explainable by the approved Spec Gate.

## Context Rule

In the `spec_gate` state, the agent may only use:

- The user's current message.
- Existing conversation context.
- Already loaded project information, if any.

The agent must not gather new project context before confirmation.

If more context is needed, write `TBD`, `Assumptions`, or `Open Questions`.

## After Confirmation

After the user confirms, respond briefly:

```md
Confirmed. I will implement only the approved Spec Gate scope and avoid unrelated changes.
```

After confirmation, the approved Spec Gate becomes the binding implementation contract for the rest of the task.

The agent must continue to apply all rules above during implementation, especially:

Anti-Drift Rules
Simplicity Rule
Context Rule

The Spec Gate is not completed merely by producing the confirmation document. It must guide all later inspection, planning, code changes, verification, and reporting.

Then proceed with normal coding-agent work:

1. Inspect only relevant files.
2. Make the smallest correct change.
3. Verify the behavior.
4. Report what changed.

## Done Report

After implementation, report briefly:

```md
# Done Report

## Summary

What changed.

## Changed Files

Files changed and why.

## Verification

What was tested.

## Not Tested

Anything not tested and why.

## Remaining Risks

Any known risk or follow-up.
```

## Trigger Examples

User:

```text
SG: Add a World Cup tab to the news page. It must support both PC and mobile. Do not touch other tabs.
```

Assistant should output a Spec Gate and wait.

User:

```text
Gate this first: clean up the audit page select-all logic. TableNo should not be displayed. Do not touch other pages.
```

Assistant should output a Spec Gate and wait.

User:

```text
Implement the approved Spec Gate.
```

Assistant may start implementation.

## Non-Goals

Do not output detailed implementation plans, API designs, file-change plans, or test commands during the gate phase unless the user explicitly asks for them.

Do not block coding agents from using their own planning, inspection, and testing ability after user approval.
