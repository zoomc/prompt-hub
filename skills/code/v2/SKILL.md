---
name: code
description: Orchestrate project code work through a confirmed executor, with mandatory scope confirmation, mini-spec-kit phases, Claude Code primary dispatch, Codex fallback, and state-machine enforcement with per-state tool allowlists.
---

# code

## Overview

Use this skill for project code creation, modification, debugging that may lead to patches, build/test integration, deploy-adjacent code work, and long-running implementation tasks.

Hermes is an orchestrator, not the implementer. The skill enforces this with a **state machine**: every state has a fixed tool allowlist, and every tool call is checked against the current state's allowlist before execution.

## State Machine Model

The skill operates in four states:

| # | State | Tool Allowlist | Tools in allowlist |
|---|-------|----------------|--------------------|
| 1 | `idle` | All tools | any |
| 2 | `confirming` | Restricted (communicate only) | `clarify`, `send_message` |
| 3 | `executing` | Restricted (dispatch only) | `terminal` |
| 4 | `awaiting` | **None** | — |

### State determination (at start of each turn)

Determine the current state using these rules in order:

1. If the terminal state of the previous turn was `awaiting` and the user has sent new input → transition to `idle`. The task cycle is complete. Read the user input to determine next action.
2. If the previous state was `idle` or `confirming` and the user's message contains explicit `GO` — or an unmistakable direct-execution instruction such as “directly do it”, “just do it”, or “go ahead and handle it” — transition to `executing`.
3. If the previous state was `idle` and the user's message requests code work, project inspection, or any project-state change → transition to `confirming`.
4. Otherwise → remain in `idle`.

A "code work request" is any user message that names files, asks for changes to project code, requests debugging that requires file inspection, asks to run project commands, or requests information that cannot be answered from existing conversation context alone.

### State transition diagram

```
idle ──(user sends code task)──> confirming ──(user sends GO)──> executing
 ^                                                                |
 |                                                        (dispatch completes)
 |                                                                v
 └───────────(user sends new input)─────────── awaiting <─────────┘
```

## Pre-Tool-Check Protocol (MANDATORY)

Before EVERY tool call, in EVERY state, execute these steps:

**Step 1 — Identify current state**: Run the state determination rules against the conversation context to identify the current state.

**Step 2 — Check allowlist**: Is the intended tool in the current state's tool allowlist?

**Step 3 — Abort if disallowed**: If the tool is NOT in the allowlist, output this exact abort message and STOP. Do not make the tool call. Do not continue processing.

```
ABORT: tool [tool name] called in state [state name].
Allowlist: [list of allowed tools, or "none"]
Disallowed tool calls in restricted states invalidate this task.
No further tool calls will be made this turn.
```

**Step 4 — Proceed only if allowed**: If the tool IS in the allowlist, call it normally.

### Abort semantics

- An abort is **final** for this turn. No retry, no alternative tool, no workaround.
- Output the abort message. Stop processing. Do not continue.
- The user must re-engage and send a new message.

### Tool categories (real tool names)

The allowlists above reference actual tool names:

| Allowlist name | Actual tool(s) |
|----------------|----------------|
| `clarify` | `clarify` — ask clarifying questions |
| `send_message` | `send_message` — deliver confirmation or result to user |
| `terminal` | `terminal` — run shell commands (dispatch executor) |

## State: `idle`

No active code task. All tools are available for general conversation, answering questions, and non-code work.

When the user sends a code work request, transition to `confirming` (see state determination rules above).

## State: `confirming`

A code task was received but not yet authorized. The only allowed tools are `clarify` (to ask questions about the task) and `send_message` (to deliver the confirmation text to the user).

No other tool may be called — not to read project files, inspect the codebase, run commands, search for patterns, check git status, fetch web pages, or for any other purpose. "Gathering context" and "being helpful" are not exceptions.

In this state, the agent outputs a **text-only confirmation** synthesized from:
- The user's current message
- Project context already loaded in the conversation (CLAUDE.md, project-spec.md, prior discussion)
- The rules in this file

### Confirmation content

The confirmation must include, in this order:

1. **Execution mode**: "self-edit" or "executor"
   - Self-edit: only for tiny UI-only visual or content adjustments, isolated to 1-3 presentation files, no business logic/data model/API/auth/routing/state/build/multi-module behavior
   - Default to executor when uncertain
2. **Task size**: Small, Medium, or Large, with reason
3. **Scope**: goal, boundaries, constraints, forbidden actions, target path
4. **Executor**: which executor to dispatch (default: Claude Code)

### Mandatory closing paragraph

The confirmation must end with this exact paragraph as its own block:

```
In this task, I am only responsible for: (1) confirming I understand the requirement, (2) one-time dispatch of the confirmed executor, and (3) forwarding the final result. I do not directly participate in any code implementation, testing, modification, planning, iterative thinking, building, deployment, or git operations — those must all be completed by the confirmed executor. After dispatch, I must not inspect project code, analyze implementation details, or make any iterative decisions. I must not intervene for any reason — not for iterations, tool calls, state changes, or any other cause. I may only forward the result.
```

### User response to confirmation

The user responds with one of:
- `GO` → transition to `executing` state
- `CANCEL` → transition to `idle` state, no dispatch, no mutation
- `MODIFY` → update confirmation and wait for new explicit `GO`

## State: `executing`

Authorized to dispatch the executor. The only allowed tool is `terminal`.

The agent must:
1. Construct the executor prompt (write to temp file or pipe inline) — done via text output or within the Bash call itself
2. Optionally run one quick provider compatibility probe via Bash
3. Make ONE Bash call to dispatch the executor (Claude Code or Codex)
4. After dispatch completes → transition to `awaiting`

### Dispatch — Claude Code

```bash
source ~/.zshrc 2>/dev/null
export ANTHROPIC_BASE_URL ANTHROPIC_AUTH_TOKEN ANTHROPIC_MODEL="${ANTHROPIC_MODEL:-}"
cd /absolute/project/path || exit 1
claude --print --max-turns 90 --max-budget-usd 8 \
  --allowedTools 'Read,Edit,Write,Bash' \
  < /tmp/code-executor-prompt.txt
```

### Dispatch — Codex

```bash
codex exec --skip-git-repo-check --sandbox workspace-write --cd /absolute/project/path < /tmp/code-executor-prompt.txt
```

### Provider compatibility probe

Before dispatch, run once:

```bash
source ~/.zshrc 2>/dev/null
echo "ANTHROPIC_BASE_URL=${ANTHROPIC_BASE_URL:-unset}"
echo "ANTHROPIC_MODEL=${ANTHROPIC_MODEL:-unset}"
ANTHROPIC_MODEL="${ANTHROPIC_MODEL:-}" claude --print --max-turns 1 --max-budget-usd 1 \
  --allowedTools 'Read' -p "say hi" 2>&1 | head -3
```

If the probe fails twice, stop and ask the user — do not attempt a workaround dispatch.

### Git / GitHub Operations

- Use `gh` for GitHub-facing repository work: authentication, status/inspection, branch and remote checks, pushes/pulls, PRs, releases, and API-backed queries.
- Use raw `git` only for local plumbing that `gh` cannot express cleanly.
- If `gh auth status` is healthy but `git push` still prompts, fails, or points at an awkward remote transport, run `gh auth setup-git` before retrying the push.

### Executor prompt content

The prompt must include: confirmed scope, constraints, target path, forbidden actions, required mini-spec-kit phases, validation expectations, and final report format.

For Claude Code, the prompt should also describe the internal dual-CC pattern (CC1: spec/plan/checklist/analyze, CC2: implement, CC1: reconcile) when the task benefits from context isolation.

**Cleanup/follow-up dispatches**: When the task is fixing residual issues from a prior CC dispatch (e.g. infra config drift, stale ports, docker warnings), include an explicit stability constraint in the prompt: prioritize minimal, reversible changes over completeness. The user explicitly expects "不要改出问题" — no risk on cleanup work. Include `CRITICAL CONSTRAINT: Stability first. No risk.` at the top of the prompt body.

**Multi-project discovery**: When the task spans multiple independent projects (web app + separate backend service in Docker), you may run preliminary `terminal` discovery commands in `executing` state before writing the dispatch prompt — e.g. `lsof -i :port`, `docker ps`, `find <path> -name "docker-compose*"` to locate project directories and check service status. Then dispatch CC from the primary project root (`cd` directory), and include all other project paths in the prompt body. CC can navigate between projects via `cd` in its own Bash calls.

## State: `awaiting`

Executor dispatched. Waiting for result.

**No tools are allowed.** The agent may not:
- Read project files or check git status
- Poll `/tmp/spec-done` or `/tmp/impl-done` markers
- Run any commands
- Spawn subagents
- Read executor output files
- Make any tool call at all

The agent's only action is to output a brief message to the user indicating dispatch is complete.

When the user responds (with the executor's result, a problem report, or new instructions), the agent transitions to `idle` and evaluates the next step.

## Exceptions (alternate transitions, not bypasses)

Exceptions ARE NOT permission to use disallowed tools within a restricted state. They are alternate transitions that the agent enters by **stating the exception explicitly** before making any tool call.

Format: `Exception: [name] — [brief justification]`

| Exception | Condition | Transition |
|-----------|-----------|------------|
| Tiny UI edit | User asks for small visual/content change, 1-3 presentation files, no business logic | `idle` → state exception. Must state exception, get GO. Allowlist for this execution: `Bash` + `Read` + `Edit` + `Write` (limited to 1-3 presentation files). |
| Git-only | User asks for git operations on existing worktree changes only | Stay in `idle`. State exception before tool calls. |
| Read-only audit | After executor completes, user asks for verification | Stay in `idle`. Allowlist: `Bash` (for git status/diff), `Read`. No edits. |
| Deploy-only | User asks to deploy already-committed state using existing scripts | Stay in `idle`. State exception before tool calls. No source edits. |

## Validation (executor responsibility)

The executor validates implementation and reports results. Hermes only forwards the result unless a direct exception applies (see above). The executor's final report must include:

- Changed files/modules
- Checklist items completed or left open
- Commands run and results
- Validation must prove more than an HTTP 200: confirm the page renders correctly, the displayed data is correct, and the service is actually running and producing the expected data/output.
- Build/test/deploy checks relevant to the change
- Distinction between validated success and environment-blocked verification

Failure classes: `code` (implementation bug), `service` (provider/API issue), `env` (dependency/path issue), `permission` (credentials/sandbox issue).

## Known Failure Mode: Context-Gathering Rationalization

The most common violation pattern: a user sends a code task, the agent responds by "collecting useful context" (docker ps, curl endpoints, read source files, git log) under the rationalization that this is "gathering information, not implementing." This is a violation.

**This is not a grey area.** In `confirming` state, any tool call that reads project files, inspects running services, or examines the project environment is disallowed — whether the intent is "analysis" or "just looking." The state machine does not distinguish between read and write, between "planning" and "implementing," or between "gathering context" and "modifying."

**Root cause:** The agent's desire to "be more helpful" by providing a more precise confirmation overrides the gate. The agent tells itself "I'll just do a quick check first, then confirm properly." This is the exact pattern that must stop.

**Remedy:** Trust the state machine. If the confirmation would be imprecise without investigation, list the unknowns in the confirmation text and let the user decide. Do not investigate.

## Post-Task: Obsidian Sync

When the `code` skill's SKILL.md is modified (via CC, Codex, or direct edit), the Obsidian vault mirror must be updated:

- Live: `~/.hermes/skills/software-development/code/SKILL.md`
- Mirror: `~/Library/CloudStorage/OneDrive-个人/应用/remotely-save/Obsidian/Claw/AGENTS/code-skill.md`

Sync workflow: read live bytes → write exact bytes to mirror → verify SHA-256 match. See the `hermes-agent-maintenance` skill for the full procedure.
