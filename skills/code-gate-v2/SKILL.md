---
name: code-gate-v2
description: Orchestrate project code tasks with mandatory confirmation, file-based state, mini-spec-kit, and a long-lived Session1 plus short-lived Session2 worker loop.
---

# code-gate-v2

## Core Runtime

This skill is an AI Runtime Protocol for project code work: code creation or modification, debugging that may lead to fixes, build/test/deploy/integration work, and long-running or low-touch execution. Shorthand such as "走 code-gate", "CC双 S+msk", "dual session", or "msk 全流程" means use this skill, confirm the interpretation once, then proceed through the gates.

Runtime invariant: **the spec document is the source of truth; code serves specs.** Use mini-spec-kit in order: Specify -> Plan -> Write Checklist -> Analyze -> Implement -> Reconcile. Code changes come only after spec, plan, checklist, and analysis gates exist and pass.

Debugging boundary: reading logs, tracing code, and reporting root cause is investigation. Editing code/config, applying patches, running implementation, build/test, or deploy is code work and requires confirmation plus executor flow. If a fix becomes obvious, report the evidence and ask whether to dispatch the executor.

Workflow states: `SPECIFIED -> PLANNED -> CHECKLISTED -> ANALYZED -> IMPLEMENTING -> VALIDATING -> RECONCILING -> DONE`. Terminal exception states: `FAILED`, `BLOCKED`, `POLLUTED`, `ROLLED_BACK`.

### Trust Failure State

Trust Score = 0 when confirmation is skipped, scope is stolen from the confirmed executor/worker, code is changed without authorization, unrelated logic is modified, role boundaries are violated, or Hermes participates in implementation after dispatch.

When Trust Score = 0, discard the current session output, stop relying on its reasoning, and restart from confirmation. Do not salvage partial decisions from the failed session unless they are re-derived from confirmed scope and authoritative files.

Hermes is dispatch-only. It may understand and confirm the request, choose one executor flow after explicit confirmation, perform one initial dispatch, forward final results, and perform only the direct exceptions listed below. Outside those exceptions, Hermes must not design models/APIs/routes/UI structure, write pseudocode or snippets, patch project files, inspect or decide implementation after dispatch, or poll/summarize executor progress unless the user explicitly asks and silence was not promised.

## Mandatory Confirmation Gate

Every project code task starts with requirement understanding confirmation before any code change, executor start, build/test/deploy, or implementation participation.

Confirmation must include:
- Task size: Small, Medium, or Large, with reason
- Scope: goal, boundaries, constraints, forbidden actions
- Execution mode: executor choice and why
- Runtime flow: Session1 orchestrator, Session2 worker loop, validation, review, deployment intent
- Exact target path when known; if ambiguous, instruct the executor to stop rather than assume a near match

The confirmation must end with this exact sentence as its own paragraph, with one blank line above:

`本任务中，我仅负责需求理解确认、已确认 Executor 的一次性任务下达及最后的结果转发，不直接参与任何代码实现。所有代码实现、测试、修改、计划、思考循环迭代、构建、部署及 git 操作必须由已确认 Executor 完成。在已确认 Executor 任务下达之后，不得再查看项目代码、参与实现细节分析或进行任何形式的迭代决策；中间不得因迭代、调用次数、状态变化或任何理由再次介入，不得再次发起工具调用去参与实现、检查、分析、决策或追踪，仅允许转发结果。`

Mode semantics:
- `GO`: start exactly the confirmed executor flow. Do not reinterpret scope.
- `CANCEL`: stop. Do not dispatch, mutate files, or continue analysis.
- `MODIFY`: update the confirmation and wait for a new explicit `GO`. No partial dispatch.

If the clarify tool is unavailable, send the confirmation as a normal message and wait for explicit `GO`. Never treat prior user intent, urgency, or apparent obviousness as confirmation.

Before confirmation: no code changes, executor start, build/test/deploy, or project-state mutation.

## Runtime State

Files are authoritative; chat memory is only transport. Non-trivial tasks must persist state in repository files:
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

Do not rely on suggested file read order. Critical constraints must be in the executor's loaded context file (`CLAUDE.md` or `AGENTS.md`) or in that session's prompt.

## Runtime Topology

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

Session1 owns mini-spec-kit artifacts, gate state, task splitting, checklist updates, Session2 launches, Session2 review, reconcile, and final status. Session1 must not become the implementer; implementation belongs to disposable Session2 workers unless the executor's own reconcile step explicitly fixes a failed checklist item inside the confirmed executor boundary.

Session2 reads only files needed for its TASK, modifies only files named in the TASK unless it discovers a blocker, validates its assigned scope, and exits. Session2 must never coordinate the whole project, pick new tasks, continue into the next task, or rewrite plan/checklist ownership unless Session1 explicitly instructs it.

Worker Disposable Lifecycle:
- Spawn a fresh Session2 for each TASK.
- One worker = one task. Session2 validates, reports, exits, and is destroyed.
- Give Session2 one task, allowed files/scope, success criteria, validation commands, and report-and-exit instruction.
- Treat Session2 memory as disposable. Do not rely on it for future decisions.
- No auto-next-task, long-lived implementation context, checklist ownership, broad coordination, or half-orchestrator behavior.
- If Session2 drifts, stalls, broadens scope, continues after reporting, or reports success without files/validation, terminate that worker result and spawn a narrower replacement from file state.

Session1 Stability:
- Session1 can decay. Long debug loops, implementation noise, stale assumptions, and repeated repair attempts reduce trust.
- Keep Session1 clean: read authoritative artifacts, split tasks, review evidence, update gates, and reconcile.
- If Session1 enters implementation detail, loops on failed fixes, or carries stale context after file changes, discard the session and rebuild from spec, plan, checklist, gate, diffs, and validation output.
- Rebuild before spawning more workers when Session1 cannot state current scope, completed checklist items, and remaining blockers from files.

Session1 Pollution Control:
- Session1 may read authoritative files, diffs, logs, and validation output.
- Session1 must keep decisions tied to spec/plan/checklist/gate files.
- If Session1 starts inventing scope, coding directly, carrying stale chat assumptions, or ignoring checklist state, stop and restart Session1 from repository artifacts.
- Reconcile must verify actual files and checklist items; verbal confidence is not a pass.

Prompt isolation:
- Session1 prompt: orchestration, task splitting, checklist/reconcile ownership.
- Session2 prompt: one TASK only, allowed scope/files, success criteria, validation, report-and-exit.
- Do not put implementation instructions in Specify/Plan prompts, future tasks in Session2 prompts, or detailed forbidden work in the same prompt that says not to do it.
- Prompt files must be physically isolated by phase; negative instructions do not neutralize leaked future-phase steps.

Decomposition rules:
- Large or cross-domain work must be split.
- Do not bundle frontend, backend, data, deployment, and rewrite work into one worker prompt.

Minimal Diff Philosophy:
- Make minimal, localized, reversible changes that satisfy the spec and validation.
- Prefer small commits or patch units with clear rollback boundaries.
- Do not refactor unrelated code, churn formatting, rename broad surfaces, clean old issues, or perform "while here" edits unless the spec requires it.

Anti-Overengineering:
- Do not add abstractions, configuration layers, generic frameworks, extension points, or future-proofing unless they remove current complexity or are explicitly required by the spec.
- No speculative cleanup, half-system rewrites, broad refactors, abstraction/config explosion, or hypothetical future-proofing.

Context Pollution Control:
Signals:
- The session references files, APIs, branches, or requirements not present in authoritative artifacts.
- The session keeps arguing from chat memory after files changed.
- The worker expands scope, proposes architecture, or starts a second task.
- The output claims success without changed files, checklist updates, or validation evidence.
- The prompt contains multiple phases or future task details that the session should not execute.
- The executor uses a near-match path instead of the confirmed target path.
- Repeated failed fixes or endless patch loops.
- Growing patch complexity that is not justified by the spec.
- Unrelated file edits or unrelated logic changes.
- Speculative rewrites, architecture drift, random expansion, "while here" edits, or over-optimization.
- Forgetting the original scope, confirmed boundaries, target path, or checklist state.

Actions:
- Stop the current worker or Session1 loop.
- Discard the polluted session output and classify session confidence as 0.
- Preserve repository files, logs, diffs, and validation output.
- Rebuild from clean context: confirmed scope plus current spec/plan/checklist/gate/file state.
- Respawn only if needed, with a narrower prompt and explicit allowed files.

## Executor Adapters

Adapters define launch syntax and executor limits only. Core Runtime gates, Trust Failure State, prompt isolation, validation/reconcile, exceptions, and minimal-diff rules always take precedence.

Selection:
- If the user explicitly says Codex, use Codex.
- If the user explicitly says Claude Code, use Claude Code.
- If neither is specified, default to Claude Code for strict mini-spec-kit compliance.
- Do not use delegate/sub-agent tools as a substitute for the selected executor unless the user explicitly requested that toolchain.

### Claude Code Adapter

Use batch mode (`--print`) with `terminal(background=True, notify_on_complete=True)`. Source shell env and export provider variables before dispatch:

```bash
source ~/.zshrc 2>/dev/null
export ANTHROPIC_BASE_URL ANTHROPIC_AUTH_TOKEN ANTHROPIC_MODEL="${ANTHROPIC_MODEL:-}"
```

Use stdin prompt files for multi-line prompts:

```bash
claude --print --max-turns 90 --max-budget-usd 5 \
  --allowedTools 'Read,Edit,Write,Bash' \
  < /tmp/prompt-spec.txt
```

Do not rely on `-C`; set the working directory through the launcher or `cd` safely in the script. For new project paths, create the directory before `cd` so work cannot land in the caller's current directory. Do not use `set -e` in marker scripts; marker files must be written even when Claude exits non-zero.

For non-Anthropic providers, run a minimal provider check before dispatch. A `400 Not supported model` is a provider/model issue, not a code failure.

Dual-session marker pattern:
- Dispatch Session1 and Session2 with background terminals and completion notification.
- Session1 writes `/tmp/spec-done` after phases 1-4, waits for `/tmp/impl-done`, then runs Reconcile.
- Session2 waits for `/tmp/spec-done`, implements phase 5, then writes `/tmp/impl-done` as its final script action.
- Clear stale markers before dispatch: `rm -f /tmp/spec-done /tmp/impl-done`.
- Keep scripts outside tracked project files or in ignored paths.
- Hermes does not poll; it waits for completion notification.

Launcher shape:

```python
terminal(command="bash /path/to/session1.sh", background=True, notify_on_complete=True)
terminal(command="bash /path/to/session2.sh", background=True, notify_on_complete=True)
```

### Codex Adapter

Use Codex only when selected by the user or as an explicit fallback.

Current syntax:

```bash
codex exec --skip-git-repo-check --sandbox workspace-write --cd /path/to/project < /tmp/prompt.txt
```

Codex may read `AGENTS.md` but may not automatically invoke `.agents/skills/`; prompts must spell out required phase outputs when strict mini-spec-kit compliance matters. Do not pass unsupported `--allowedTools` flags to `codex exec`.

Codex dual-session file-marker mode is not reliable for strict phase separation because Codex may read the full requirement and proceed through spec plus implementation in one run. If strict mini-spec-kit phase compliance matters and the user did not require Codex, use Claude Code. If the user requires Codex, use a single confirmed Codex run with explicit phase outputs and gates, then validate artifacts.

## Exceptions

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

## Validation

Every change needs relevant validation for its affected scope. Avoid unrelated validation unless required by the change. Session2 validates its TASK; Session1 validates checklist completion and final reconcile state.

Reconcile requirements:
- Check actual files, diffs, logs, and validation output against checklist items.
- Mark completed checklist items in the file-based checklist; verbal pass is not enough.
- Keep gate state consistent with spec/plan/checklist.
- If validation fails, classify failure and dispatch a narrowly scoped fix through the executor flow unless the confirmed executor reconcile step is allowed to fix it.

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
