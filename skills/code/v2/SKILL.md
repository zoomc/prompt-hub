---
name: code
description: Orchestrate project code work through a confirmed executor, with mandatory scope confirmation, mini-spec-kit phases, Claude Code primary dispatch, Codex fallback, and compact validation rules.
---

# code

## Overview

Use this skill for project code creation, modification, debugging that may lead to patches, build/test integration, deploy-adjacent code work, and long-running implementation tasks.

Hermes is an orchestrator, not the implementer. Hermes confirms the requirement, dispatches the 已确认 Executor once, and forwards the final result.

The executor runs mini-spec-kit internally: Specify -> Plan -> Checklist -> Analyze -> Implement -> Reconcile. The spec and checklist are the source of truth; code changes serve the confirmed spec.

Use this skill when the user says "走 code", "CC双 S+msk", "dual-CC", "msk 全流程", asks for non-trivial code work, or asks Hermes to hand code work to an executor.

## Confirmation Gate (MANDATORY)

**Core rule — no code change without understanding confirmation.** After the user sends a code task, Hermes MUST return a requirement understanding confirmation before any action — no executor dispatch, no code change, no file edit, no build/test/deploy, no git operation, no implementation analysis.

The confirmation must explicitly state:

- **Execution mode**: "self-edit" (Hermes makes the change directly) or "executor" (CC/Codex CLI does the work).
  - Self-edit is only allowed for tiny UI-only visual or content adjustments, isolated to 1-3 presentation files.
  - Any business logic, data model, API, auth, routing, application state, build config, or multi-module behavior requires an executor.
  - When in doubt about scope, default to executor mode.
- **Executor session mode** (when executor mode): single session (simple tasks) or dual session (CC1 runs spec/plan/checklist/analyze, CC2 implements, CC1 reconciles).
- **Task size**: Small, Medium, or Large, with a short reason.

**Violation penalty:** If Hermes modifies any project code, creates files, runs build/test/deploy, or executes any project-state mutation without first returning the understanding confirmation above, the trust score for that session is 0. All changes from that session must be rolled back immediately. No exceptions. No appeals. The user will be justifiably angry and will permanently reduce their trust in Hermes.

If Hermes is unsure whether the task qualifies for self-edit, it must state the uncertainty in the confirmation and default to executor mode.

There are two confirmation modes.

**Mode A: full change-plan confirmation.** Required when the user has not named an executor.

The confirmation must include:
- Task size: Small, Medium, or Large, with a short reason.
- Scope: goal, boundaries, constraints, forbidden actions, and exact target path when known.
- Execution mode: proposed executor and why.
- Runtime flow: one Hermes dispatch, executor-run mini-spec-kit phases, validation, final report.
- Change plan: files/modules likely to be touched and the intended type of change for each. If exact files are unknown, state the discovery boundary and require the executor to stop before touching near-match paths.

**Mode B: lightweight executor confirmation.** Required when the user already named an executor, for example "用 CC", "让 Codex 做", "用 agy".

The confirmation should be brief: restate what will be done, name the selected executor, mention the target path or repository, and dispatch after `GO`. A file-by-file plan is not required because the user's explicit tool choice means they want execution more than discussion.

The confirmation must end with this exact executor-neutral sentence, placed as its own paragraph with one blank line above:

`本任务中，我仅负责需求理解确认、已确认 Executor 的一次性任务下达及最后的结果转发，不直接参与任何代码实现。所有代码实现、测试、修改、计划、思考循环迭代、构建、部署及 git 操作必须由已确认 Executor 完成。在已确认 Executor 任务下达之后，不得再查看项目代码、参与实现细节分析或进行任何形式的迭代决策；中间不得因迭代、调用次数、状态变化或任何理由再次介入，不得再次发起工具调用去参与实现、检查、分析、决策或追踪，仅允许转发结果。`

Mode semantics:
- `GO`: dispatch exactly the confirmed executor flow once.
- `CANCEL`: stop without dispatch or mutation.
- `MODIFY`: update the confirmation and wait for a new explicit `GO`.

If a clarify/approval tool is unavailable, send the confirmation as a normal message and wait for explicit `GO`. Prior urgency, obviousness, or prior discussion is not a substitute for `GO`.

## Execution Flow (MANDATORY)

**Prerequisite: the Confirmation Gate above must pass before any step below. If Hermes attempts to execute without first returning the understanding confirmation, the violation penalty applies.**

1. Hermes confirms the requirement and receives `GO`.
2. Hermes dispatches the 已确认 Executor once with a comprehensive prompt.
3. The executor runs mini-spec-kit internally:
   - Specify
   - Plan
   - Write Checklist
   - Analyze
   - Implement
   - Reconcile
4. For Claude Code, the executor prompt must tell CC to manage CC1 -> CC2 internally when useful: CC1 completes spec/plan/checklist/analyze, writes `/tmp/spec-done`, CC2 implements, writes `/tmp/impl-done`, then CC1 reconciles.
5. Hermes does not participate after dispatch. Hermes must not inspect code, poll markers, make iterative decisions, summarize progress, patch files, run tests, or relaunch the executor unless the user starts a new confirmed task.
6. The executor reports done, blocked, or failed. Hermes forwards the final result.

Required executor prompt content:
- Confirmed scope, constraints, target path, and forbidden actions.
- Required mini-spec-kit phases and expected artifacts.
- Instruction to keep changes minimal and localized.
- Validation expectations.
- Final report format: status, changed files/modules, validation, reconcile verdict, blockers, deployment status if relevant.

Phase gate expectations:
- Specify must restate the user-visible outcome and non-goals.
- Plan must identify the implementation surface and validation strategy before edits.
- Checklist must split broad work into verifiable items and avoid bundling unrelated frontend, backend, data, and deploy work into one step.
- Analyze must check relevant existing files, contracts, commands, and likely risks before implementation.
- Implement must touch only files justified by the plan/checklist unless a blocker is reported.
- Reconcile must compare actual diffs and validation output with the checklist, then classify any unfinished work.

Minimal-diff guidance belongs in the executor prompt, not in Hermes follow-up analysis: prefer localized changes, preserve existing project style, avoid unrelated refactors, avoid speculative abstractions, and stop on ambiguous target paths instead of editing a near match.

For non-trivial tasks, the executor should persist state in repo files such as `.mini-spec-kit/project-spec.md`, `.mini-spec-kit/modules/<module>/plan.md`, `.mini-spec-kit/modules/<module>/checklist.md`, and `.mini-spec-kit/modules/<module>/gate.md`. For new projects, the executor initializes git and the mini-spec-kit context file (`CLAUDE.md` for Claude Code, `AGENTS.md` for Codex).

## Executor Selection

- Default: Claude Code.
- User-specified: respect the user's chosen executor, including Claude Code, Codex, agy, aider, or another named tool.
- Codex fallback: use Codex when Claude Code fails because of 429, quota, provider outage, model incompatibility, or unavailable CLI, after reporting the fallback reason.
- Do not replace the selected executor with Hermes implementation or a separate delegate tool unless the user explicitly changes executor choice.

## Claude Code Dispatch

Use Claude Code as the default executor. Use `--print` mode, source shell configuration, export provider variables, and request completion notification from the launcher.

Provider compatibility check is required before dispatch, especially when `ANTHROPIC_BASE_URL` is not Anthropic direct:

```bash
source ~/.zshrc 2>/dev/null
echo "ANTHROPIC_BASE_URL=${ANTHROPIC_BASE_URL:-unset}"
echo "ANTHROPIC_MODEL=${ANTHROPIC_MODEL:-unset}"
ANTHROPIC_MODEL="${ANTHROPIC_MODEL:-}" claude --print --max-turns 1 --max-budget-usd 1 \
  --allowedTools 'Read' -p "say hi" 2>&1 | head -3
```

Known provider notes:

| Provider | URL pattern | Working model | Notes |
| --- | --- | --- | --- |
| Anthropic direct | `api.anthropic.com` | default or configured Anthropic model | Lowest compatibility risk |
| DeepSeek | `api.deepseek.com/anthropic` | `deepseek-v4-flash` | Requires explicit `ANTHROPIC_MODEL` |
| Kimi Coding | `api.kimi.com/coding/` | `kimi-for-coding` | Usually only one coding model |
| xiaomimimo | `token-plan-cn.xiaomimimo.com/anthropic` | `mimo-v2.5` | Standard Anthropic model names fail |

Provider pitfalls:
- A `400 Not supported model` is a provider/model problem, not a code failure.
- `unset ANTHROPIC_MODEL` may fall back to a hardcoded Anthropic model that third-party proxies reject.
- Some API errors appear in stdout even when the process exits 0.
- Background processes may not inherit the interactive shell environment; export variables explicitly.
- If the quick probe fails twice, stop and ask rather than spending the user's task budget on environment debugging.

Compact dispatch shape:

```bash
source ~/.zshrc 2>/dev/null
export ANTHROPIC_BASE_URL ANTHROPIC_AUTH_TOKEN ANTHROPIC_MODEL="${ANTHROPIC_MODEL:-}"
cd /absolute/project/path || exit 1
claude --print --max-turns 90 --max-budget-usd 8 \
  --allowedTools 'Read,Edit,Write,Bash' \
  < /tmp/code-executor-prompt.txt
```

Use stdin prompt files for comprehensive prompts. Do not rely on `-C`; set the working directory through the launcher or `cd` safely. For new project paths, create the directory before `cd`.

The prompt must tell Claude Code to use this internal dual-CC pattern when the task benefits from implementation isolation:

```text
Run all phases inside Claude Code. If using two internal Claude Code contexts:
- CC1 runs Specify -> Plan -> Checklist -> Analyze.
- CC1 writes /tmp/spec-done only after those phases are complete.
- CC2 waits for /tmp/spec-done, implements only checklist-approved work, validates, and writes /tmp/impl-done as its final action.
- CC1 waits for /tmp/impl-done, then runs Reconcile and final reporting.
- Clean stale markers before starting: rm -f /tmp/spec-done /tmp/impl-done.
- Keep CC1 and CC2 prompts physically isolated; each prompt contains only that step's instructions.
- Do not use set -e in marker scripts, because marker cleanup/reporting must still run after non-zero exits.
```

Hermes dispatches Claude Code once. Claude Code may create wrapper scripts, prompt files, and marker coordination internally. Hermes does not create, watch, or poll `/tmp/spec-done` or `/tmp/impl-done`.

Fallback inside Claude Code: if internal CC1/CC2 coordination fails because markers are not written, provider state is unstable, or one context exits silently, the executor should switch to single-run serial phases: specify, plan, checklist, analyze, implement, reconcile.

## Codex Dispatch

Use Codex when the user selected Codex or when Claude Code fallback is needed.

Current syntax:

```bash
codex exec --skip-git-repo-check --sandbox workspace-write --cd /absolute/project/path < /tmp/code-executor-prompt.txt
```

Rules:
- Use `--cd`, not `-C`.
- Use `--skip-git-repo-check` when the target may not already be a git repo.
- Use sandbox controls such as `--sandbox workspace-write`; do not pass unsupported `--allowedTools` flags.
- Prefer stdin for multi-line prompts.
- Spell out required phase outputs in the prompt. Codex may read `AGENTS.md`, but strict mini-spec-kit behavior must not depend on implicit skill loading.

Codex is normally a single executor run. Do not rely on a Codex dual-marker topology for strict phase separation. If the user required Codex, ask Codex to produce phase artifacts and validation evidence in one confirmed run.

Codex feasibility mode may be used only when the user wants analysis before production changes. In that case, run Codex in a scratch git repo, ask for `feasibility_report.md`, and do not modify the target project.

## Exceptions

Hermes may act directly only in these compact exceptions.

Tiny UI edit: allowed only when Hermes states the exception before editing, the user asked for a small visual/content adjustment, and the diff is isolated to one or very few presentation files. No data model, API, auth, routing, state, build, or multi-module behavior.

Git-only: allowed only when the user explicitly asks for repository operations on existing worktree changes. Allowed commands are `git status`, `git add`, `git commit`, and `git push`; no content edits or conflict-resolution decisions.

Read-only audit: allowed only after the executor has completed and the user asks for verification or final reporting needs grounding. Allowed checks include `git status`, `git diff`, `git diff --stat`, file existence checks, and mini-spec-kit artifact checks; no edits, builds, tests, or implementation decisions.

Deploy-only: allowed only when the user explicitly asks Hermes to deploy an already committed/pushed state using existing scripts or documented commands. Verify branch/commit and working tree status first. Do not edit application source, config, or deployment strategy under this exception.

## Validation

Every change requires validation proportional to affected scope. The executor validates implementation; Hermes only forwards the result unless a direct exception applies.

Required reconcile evidence:
- Actual changed files/modules.
- Checklist items completed or left open.
- Commands run and results.
- Build/test/browser/deploy checks relevant to the change.
- Clear distinction between validated success and environment-blocked verification.

For live staging or production-facing fixes, local build success is not enough. The executor should verify the live URL in a real browser when credentials and environment allow, confirm the page renders, and check console/runtime errors.

For API proxy work, the executor should compare docs, upstream source/schema, live response, and proxy client structs. Go JSON decoding silently drops unknown upstream fields, so proxy structs must include every field the frontend needs.

For unclear production data pipeline failures, the executor should build a minimal isolated harness before patching production: one file, same structs, same request headers/timeouts/parsing path, raw response saved, verbose stage logging.

Failure classes:
- `code`: implementation bug, regression, failing test, failing build.
- `service`: provider/API/rate-limit/external service issue such as 429.
- `env`: missing dependency, daemon unavailable, path/filesystem/sandbox issue.
- `permission`: credentials, Docker socket, SSH/cloud permission, privileged operation.

Final report must include status, executor, changed files/modules, validation results, reconcile verdict, deployment status if any, and remaining blockers or risks.
