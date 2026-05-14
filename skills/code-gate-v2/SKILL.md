---
name: code-gate-v2
description: Orchestrate project code tasks with mandatory confirmation, file-based state, mini-spec-kit, and a long-lived Session1 plus short-lived Session2 worker loop.
---

# code-gate-v2

## Core Policy

**Spec-driven development: the spec document is the source of truth, code serves specs.**

Use this skill for project code work: code creation or modification, debugging that may lead to fixes, build/test/deploy/integration work, and long-running or low-touch execution. Shorthand such as "走 code-gate", "CC双 S+msk", "dual session", or "msk 全流程" means use this skill, confirm the interpretation once, then proceed through the gates.

Follow mini-spec-kit in order: Specify -> Plan -> Write Checklist -> Analyze -> Implement -> Reconcile. Code changes come only after the spec, plan, checklist, and analysis gates are in place. Files are authoritative; chat memory is only a transport layer. Urgency or casual wording never weakens the gate.

Hermes is the dispatcher, not the architect or implementer. It may understand and confirm the request, choose and dispatch the executor flow once after explicit confirmation, forward final results, and perform only the exceptions listed below. Outside those exceptions, Hermes must not design models/APIs/routes/UI structure, write pseudocode or snippets, patch project files, inspect or decide implementation after dispatch, or poll/summarize executor progress unless the user explicitly asks and silence was not promised.

Debugging boundary: reading logs, tracing code, and reporting root cause is investigation. Editing code/config, applying patches, or running implementation is code work and requires confirmation plus executor flow. If a fix becomes obvious, report the evidence and ask whether to dispatch the executor.

Violation of the role boundary or confirmation gate invalidates the run: confidence = 0; restart from confirmation.

## Mandatory Confirmation Gate

Every project code task starts with requirement understanding confirmation before any code change, executor start, build/test/deploy, or implementation participation.

Confirmation must include:
- Task size: Small, Medium, or Large, with reason
- Scope: goal, boundaries, constraints, forbidden actions
- Execution mode: executor choice and why
- Flow: Session1 orchestrator, Session2 worker loop, validation, review, deployment intent
- Exact target path when known; if ambiguous, instruct the executor to stop rather than assume a near match

The confirmation must end with this exact sentence as its own paragraph, with one blank line above:

`本任务中，我仅负责需求理解确认、Codex CLI 任务初始的一次性下达及最后的结果转发，不直接参与任何代码实现。所有代码实现、测试、修改、计划、思考循环迭代、构建、部署及 git 操作必须由 Codex CLI 完成。在 Codex CLI 任务下达之后，不得再查看项目代码、参与实现细节分析或进行任何形式的迭代决策；中间不得因迭代、调用次数、状态变化或任何理由再次介入，不得再次发起工具调用去参与实现、检查、分析、决策或追踪，仅允许转发结果。`

If the clarify tool is unavailable, send the confirmation as a normal message and wait for an explicit "go". Never treat prior user intent as confirmation.

Before confirmation: no code changes, executor start, build/test/deploy, or project-state mutation.

## File Source of Truth

For non-trivial tasks, state must be persisted in repository files. Use mini-spec-kit artifacts such as:
- `.mini-spec-kit/project-constraints.md`
- `.mini-spec-kit/project-spec.md`
- `.mini-spec-kit/modules/<module>/spec.md`
- `.mini-spec-kit/modules/<module>/plan.md`
- `.mini-spec-kit/modules/<module>/checklist.md`
- `.mini-spec-kit/modules/<module>/gate.md`
- logs, status files, patches, and validation output when useful

Executor context files:

| Agent | Context file | Command / skill style |
| --- | --- | --- |
| Claude Code | `CLAUDE.md` | `.claude/commands/*.md` |
| Codex | `AGENTS.md` | `.agents/skills/speckit-*/SKILL.md` |

Before dispatch, ensure the chosen executor has the matching context file and that it contains the mini-spec-kit workflow table. If missing, the executor must initialize it; Hermes does not create project files directly.

For new projects, the executor must initialize mini-spec-kit:
1. Create the project directory and run `git init`.
2. Prefer local `/Volumes/ExSSD/projects/mini-spec-kit`; otherwise clone `https://github.com/zoomc/mini-spec-kit`.
3. Create/update `CLAUDE.md` or `AGENTS.md` with the workflow table.
4. Run Specify -> Plan -> Checklist -> Analyze before code changes.

Do not rely on a suggested file read order. Critical constraints must be in the executor's loaded context file (`CLAUDE.md` or `AGENTS.md`) or in that session's prompt.

## Execution Model

Use a long-lived Session1 as the mini-spec-kit orchestrator and repeatable short-lived Session2 workers, each responsible for exactly one small TASK.

Flow:
1. Hermes confirms the requirement and dispatches Session1.
2. Session1 runs Specify -> Plan -> Checklist -> Analyze.
3. Session1 selects exactly one small, verifiable, single-domain TASK from the checklist/plan.
4. Session1 spawns Session2 with only that TASK, permitted scope/files, success criteria, validation, and report-and-exit instruction.
5. Session2 implements only that TASK, validates it, reports changed files, validation, and blockers, then exits.
6. Session1 reviews files, diffs, logs, validation output, and checklist state.
7. Session1 marks completed checklist items, splits broad work if needed, selects the next TASK, and spawns a new Session2.
8. Repeat until all checklist items are done.
9. Session1 runs Reconcile, classifies failures, and produces the final report.

Session1 owns mini-spec-kit artifacts, gate state, task splitting, checklist updates, Session2 launches, Session2 review, reconcile, and final status.

Session2 reads only files needed for its TASK, modifies only files named in the TASK unless it discovers a blocker, validates its assigned scope, and exits. Session2 must never coordinate the whole project, pick new tasks, continue into the next task, or rewrite plan/checklist ownership unless Session1 explicitly instructs it.

Prompt isolation:
- Session1 prompt: orchestration, task splitting, checklist/reconcile ownership.
- Session2 prompt: one TASK only, allowed scope/files, success criteria, validation, report-and-exit.
- Do not put implementation instructions in Specify/Plan prompts, future tasks in Session2 prompts, or detailed forbidden work in the same prompt that says not to do it.

Decomposition rule: large or cross-domain work must be split. Do not bundle frontend, backend, data, deployment, and rewrite work into one worker prompt.

## Executor Selection

Selection:
- If the user explicitly says Codex, use Codex.
- If the user explicitly says Claude Code, use Claude Code.
- If neither is specified, default to Claude Code for strict mini-spec-kit compliance.
- Do not use delegate/sub-agent tools as a substitute for the selected executor unless the user explicitly requested that toolchain.

Claude Code:
- Use batch mode (`--print`) with `notify_on_complete=True`.
- Source shell env and export `ANTHROPIC_BASE_URL`, `ANTHROPIC_AUTH_TOKEN`, and `ANTHROPIC_MODEL` when needed.
- Use stdin prompt files for multi-line prompts.
- Do not rely on `-C`; set the working directory through the launcher or `cd` safely in the script.
- For non-Anthropic providers, run a minimal provider check before dispatch. A `400 Not supported model` is a provider/model issue, not a code failure.

Codex:
- Use Codex only when selected by the user or as an explicit fallback.
- Current syntax: `codex exec --skip-git-repo-check --sandbox workspace-write --cd /path/to/project < /tmp/prompt.txt`.
- Codex may read `AGENTS.md` but may not automatically invoke `.agents/skills/`; prompts must spell out required phase outputs when strict mini-spec-kit compliance matters.
- Do not pass unsupported `--allowedTools` flags to `codex exec`.

## Direct Exceptions

Hermes may act directly only in these cases.

Tiny UI-only direct edit:
- Allowed only when Hermes states it is using the exception before editing, the user asks for a small visual/content/UI adjustment, the diff is isolated to one or very few presentation files, and no data model/API/auth/state/build/multi-module behavior is involved.
- Examples that may qualify: copy tweak in one component, CSS spacing/color adjustment in one file, replacing an icon or label in an isolated view.
- Examples that do not qualify: new behavior, multi-component layout rewrite, business logic, routing, backend, database, auth, payment, deployment, test infrastructure, unclear requirements, or validation requiring broader project reasoning.
- When in doubt, use the full Session1 + Session2 flow.

Git-only:
- Allowed only when the user explicitly asks Hermes to perform only repository operations on already-existing worktree changes.
- Allowed commands: `git status`, `git add`, `git commit`, `git push`.
- No file edits, merge-conflict content resolution, implementation decisions, or architecture decisions.
- If auth, upstream, permissions, or conflicts block git, stop and report.

Post-executor read-only audit:
- Allowed only after the executor has fully completed and the user asks or final reporting needs grounding.
- Allowed checks: `git status`, `git diff --stat`, `git diff --name-only`, `git diff --name-status`, `git diff`, and file existence/status checks for `spec.md`, `plan.md`, `checklist.md`, `gate.md`, logs, and expected artifacts.
- No edits, builds/tests, new executor loop, or implementation decisions.
- If the audit finds missing mini-spec-kit artifacts, suspicious diffs, or unverified claims, report the gap and ask whether to audit deeper, revert, or dispatch a new confirmed task.

Deploy-only:
- Allowed only when the user explicitly asks Hermes to deploy an already-committed/pushed state.
- Before deploy: verify target branch/commit and working tree status. If dirty and the user still wants deploy, preserve unrelated changes with `git stash push -u -m <tag>` before `git pull --ff-only`.
- During deploy: use existing deploy scripts, Docker/compose commands, SSH/cloud CLI commands, and repo-defined health checks. Do not edit code/config or invent a deployment strategy.
- After deploy: verify live service with `docker compose ps` or repo-specific health checks. For pull-based deploys, confirm `HEAD == origin/<branch>`.
- If deployment fails, classify as `code`, `env`, `service`, or `permission`; stop and report the safest next option.

## Validation and Reporting

Every change needs relevant validation for its affected scope. Avoid unrelated validation unless required by the change. Session2 validates its TASK; Session1 validates checklist completion and final reconcile state.

Failure classification:
- `code`: bug, regression, failing test/build caused by implementation
- `service`: API/provider/rate-limit/external service failure such as 429
- `env`: sandbox, missing local dependency, daemon unavailable, filesystem/path issue
- `permission`: blocked Docker socket, privileged port, host-only daemon, credentials, cloud/SSH permission

Do not claim runtime success when the environment blocks runtime verification. Separate what changed, what was validated, and what remains blocked.

Final report must include:
- Status: done / not done
- Execution mode and executor
- Changed files/modules
- Validation results
- Checklist/reconcile verdict
- Deployment status, if any
- Remaining risks or blockers

Silent completion:
- If the user asks for no intermediate updates, dispatch once and report only final completion or real blockers.
- If the toolchain cannot suppress intermediate status, disclose that limitation before starting.
