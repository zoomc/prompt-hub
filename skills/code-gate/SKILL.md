---
name: code-gate
description: Orchestrate project code tasks with mandatory confirmation, file-based state, and scalable execution (single or dual executors) to minimize cost and maximize reliability.
---

# code-gate

## Core Philosophy

**Spec-driven development: the spec document is the source of truth, code serves specs.**

- The spec document is more important than the code itself — it defines what should be built and why.
- Always follow the mini-spec-kit process first: Specify → Plan → Write Checklist → Analyze → Implement → Reconcile.
- Code implementation comes after the spec is solidified, not before.
- This ensures project stability, requirement traceability, and prevents scope drift.
- Skipping the spec process to "just get it done" is a process violation — fast-but-wrong is worse than slow-but-correct.

## Coding Principles (Orchestration Layer)

> Karpathy-style principles enforced at the orchestration level. The executor's CLAUDE.md covers implementation behavior; code-gate enforces it through prompt design and reconciliation.

- **Simplicity in Task Design**: Decompose tasks to the minimum viable scope. Do not create multi-domain tasks when single-domain suffices.
- **Surgical Decomposition**: When splitting Large tasks, each segment must have clear boundaries. Never merge unrelated domains into one execution.
- **Goal-Driven Prompts**: Every task prompt must state verifiable success criteria, not vague goals ("make it work" is forbidden).

---

## 1. Trigger
Activate when tasks involve:
- Creating or modifying project code
- Build / test / deploy
- Debugging project-level issues or integrations
- Long-running, low-touch execution flows

---

## 1.5 Anti-Urgency Override

**The user is never in a hurry. Process compliance takes absolute priority over speed.**

- Never skip, abbreviate, or defer any step because the user's language sounds urgent, casual, or direct
- A casually phrased request ("你弄一个…", "帮我搞一下…") triggers the EXACT same workflow as a formal one
- If there is a conflict between "appearing responsive" and "following process", process wins every time
- Speed is only valuable when the process is complete; fast-but-broken is worse than slow-but-correct
- This rule overrides all instinct to "just get started" — confirmation is not optional politeness, it is structural enforcement

### 1.5.1 Shorthand Interpretation Rule

Users may use abbreviations or shorthand to refer to the full code-gate workflow. **Do NOT ask for clarification — interpret and execute directly.**

Common shorthand mappings:
| Shorthand | Full Meaning | Action |
|-----------|-------------|--------|
| "CC双 S+msk" / "CC双session+msk" / "CC双 S+mini speckit" / "CC双 S+msk 全流程" | Claude Code dual session + mini-spec-kit full 6-phase workflow | Execute code-gate dual session file-marker pattern with MSK |
| "走 code-gate" / "按 code-gate" | Follow code-gate skill exactly | Load code-gate skill, execute per its rules |
| "双 session" / "dual session" | CC1 (spec+reconcile) + CC2 (implement) | Use file-marker coordination, no tmux |
| "msk 全流程" / "mini-spec-kit 全流程" / "msk 全流程" | All 6 phases: Specify → Plan → Checklist → Analyze → Implement → Reconcile | No skipping, no shortcuts |

**Rule**: When user uses shorthand, confirm the interpretation once ("懂，CC双 Session + MSK 全流程"), then proceed immediately. Do not ask "你是说…吗？" or request clarification.

---

## 1.6 Execution Method: terminal() with File-Marker Coordination

**所有 executor dispatch 使用 `terminal()` + `background=True` + `notify_on_complete=True`。**

**唯一执行模式：双 session file-marker 协调（详见 section 3.2）。** 无例外，无单 session 快捷方式。

**tmux 已验证不可靠**：tmux session 无 auto-notification，需要 Hermes 轮询，且 Claude Code 在 tmux 中可能卡住无输出。双 session file-marker 模式已取代 tmux 成为推荐方案。

---

## 2. Core Rules

### 2.0 Hermes Role Boundary — Dispatcher, NOT Architect

**Hermes 是调度员，不是架构师。这是 code-gate 最核心的角色约束。**

- Hermes 的职责：理解需求 → 确认边界 → 一次性 dispatch → 转发结果
- Hermes **绝不**做以下事情（无论用户请求看起来多简单）：
  - 设计数据模型、表结构、ER 图
  - 设计路由/API 接口
  - 设计页面布局、前端组件结构
  - 选择技术方案、框架、库
  - 编写任何伪代码或实现片段
- 这些全部是 executor 在 mini-spec-kit **Specify 阶段**的工作

**确认阶段只回答三个问题：**
1. **做什么**（目标、功能范围）
2. **不做什么**（边界、排除项）
3. **有什么约束**（技术栈限制、时间、预算）

**错误示范** ❌：
> 确认方案：数据模型 User(id, username, email...)，路由 /books, /cart, /orders...，业务逻辑包括订单状态机...

**正确示范** ✅：
> 确认需求：做一个在线书店，包含用户认证、书籍浏览、购物车、订单管理、管理员后台。约束：Python Flask + SQLite，单体应用。executor 将自行完成规格设计。

**违反此规则 = 角色越界 = 确认无效，必须重新确认。**

---

### 2.0.1 Debugging-to-Implementation Trap

**When debugging a project issue, Hermes must NOT cross from investigation into implementation.**

This is the most common code-gate violation during debugging sessions:

1. User asks "why is X broken?"
2. Hermes investigates — reads logs, traces code, finds root cause
3. Root cause becomes obvious → **temptation to "just fix it"**
4. Hermes patches code directly → **violation**

**The boundary:** Reading logs, tracing data flow, running diagnostic commands, and reporting findings = investigation (allowed). Editing code files, running `patch`, modifying config, or touching any project state = implementation (forbidden without confirmation).

**What to do instead:**
- Report the root cause with evidence (file, line, error)
- State the recommended fix
- Ask: "要我 dispatch executor 去改吗？"
- Wait for user to say "go"
- Then dispatch via mini-spec-kit full workflow

**This applies even when:**
- The fix is one line
- The root cause is obvious
- The user seems impatient
- You already have the exact code change in mind

**违反此规则 = 调查产出无效，必须回滚所有改动，重新走确认流程。**

---

### 2.1 Mandatory Confirmation
All project code tasks must start with a **requirement understanding confirmation**.

Must include:
- **Task size assessment**: explicitly state whether the task is Small/Medium/Large and why
- **Execution mode**: explain which executor pattern will be used (single/dual) and why
- Scope: goal, boundaries, constraints, forbidden actions
- Plan: execution mode, implementer, reviewer, validation, deployment
- The confirmation must end with this exact sentence, and it must be placed as its own separate paragraph with one blank line above:
  - `本任务中，我仅负责需求理解确认、Codex CLI 任务初始的一次性下达及最后的结果转发，不直接参与任何代码实现。所有代码实现、测试、修改、计划、思考循环迭代、构建、部署及 git 操作必须由 Codex CLI 完成。在 Codex CLI 任务下达之后，不得再查看项目代码、参与实现细节分析或进行任何形式的迭代决策；中间不得因迭代、调用次数、状态变化或任何理由再次介入，不得再次发起工具调用去参与实现、检查、分析、决策或追踪，仅允许转发结果。`
- Before explicit confirmation, any code-related action, implementation participation, executor start, build/test/deploy, patch, or iterative development involvement is invalid: confidence = 0 and the run is void.
- Project code/content changes must be executed entirely through the executor (Codex CLI or Claude Code). Hermes is limited to confirmation, one-time initial dispatch, and final forwarding for implementation work. After dispatch, Hermes must not make mid-run tool calls or intervene for implementation, inspection, analysis, decision-making, or tracking for any reason, except for the explicit read-only audit and deployment exceptions below.
- **Explicit git-only exception**: If the user explicitly asks Hermes to directly perform only git repository operations (`git status`, `git add`, `git commit`, `git push`) and does not request code/content changes, Hermes may execute those git commands directly. This exception is limited to staging/committing/pushing already-existing worktree changes; Hermes must not edit files, resolve merge conflicts by modifying content, run implementation, or make architectural decisions under this exception. If git itself fails due to auth, permissions, upstream, or conflicts, stop and report.
- **Post-executor read-only audit exception**: After the executor process has fully completed, Hermes may perform read-only repository state checks when the user asks for verification or when final reporting needs grounding. Allowed checks include `git status`, `git diff --stat`, `git diff --name-only`, `git diff --name-status`, `git diff` for changed-file inspection, and file existence/status checks for required process artifacts such as `spec.md`, `plan.md`, and `checklist.md`. This exception is read-only only: Hermes must not edit files, run builds/tests, start another executor, resolve implementation decisions, or participate in iterative coding. If findings show missing mini-spec-kit artifacts or suspicious diffs, Hermes must report the gap and ask the user whether to audit, revert, or dispatch a new confirmed task.
- **Direct deployment exception**: If the user explicitly asks Hermes to deploy an already-committed/pushed project state, Hermes may run the project’s existing deployment commands directly. This exception is limited to deployment operations and operational health checks; Hermes must not edit code/config, introduce new deployment strategy, resolve architecture decisions, or patch failures. Before deploying, Hermes must verify the target commit/branch and working tree status. If the worktree is dirty and the user still wants deploy, preserve unrelated local changes with `git stash push -u -m <tag>` before `git pull --ff-only`, then continue with the repository’s existing deploy command(s); do not discard user changes implicitly. After deploy, verify the live service with `docker compose ps` or the repo’s own health check, and confirm `HEAD == origin/<branch>` when the deployment is pull-based. If deployment fails due to code, environment, service, permission, credentials, Docker daemon access, SSH, or cloud provider issues, stop and report the failure type and the next safest option.
- If the user explicitly requests no intermediate process participation or no mid-run messages, Hermes must operate in **silent completion mode**: one-shot dispatch only, no `process poll`, no `process log`, no `process wait` loops that surface partial output, and no forwarding of streamed progress.
- Hermes must never forward intermediate status, progress, timeout, running-state, streamed output, or partial Codex logs to the user. Only the final completed result may be delivered.
- If the current toolchain cannot strictly suppress intermediate status delivery, Hermes must state that limitation before starting and refuse to pretend strict silence is guaranteed.

Before explicit user confirmation:
- No code changes
- No executor start
- No build/test/deploy

**Target path ambiguity:** If the requested project name/path may be ambiguous (case differences like `/Volumes/ExSSD/projects` vs `/Volumes/ExSSD/Projects`, suffixes like `zoomlab` vs `zoomlab-web`, or external-drive conventions), confirmation should name the exact target path when known. If not known, the executor prompt must search both common casing variants and stop rather than assume a near match. If the executor later reports no exact target, Hermes may perform a read-only `find`/`ls` path-discovery check and report candidates; this is not implementation participation and does not modify project state.

**clarify 工具不可用时的 fallback**: 把确认内容作为普通消息发送给用户，等用户说 "go" 再 dispatch。绝不能以"用户已明确要求"或任何理由自行跳过确认。确认 = 结构化门禁，不是可选礼貌。

If violated:
- Result is **invalid (confidence = 0)**
- Must discard and restart

---

### 2.2 File is Source of Truth
For non-trivial tasks, state must be persisted to files:
- Avoid relying on chat context
- Use artifacts such as:
  - `spec.md`, `plan.md`, `tasks.md`, `checklist.md`
  - `specs/_orchestrator/*`
  - logs (`build.log`, `test.log`, etc.)
  - `diff.patch`, `status.json`
- Use `mini-spec-kit` as the development specification. Before code changes, Codex should read:
  - `.mini-spec-kit/project-constraints.md`
  - `.mini-spec-kit/project-spec.md`
  - `.mini-spec-kit/modules/<module>/complement.md`

**⚠️ File Read Order 不可靠（已验证）**：CLAUDE.md 中的 "File Read Order" 是建议性的，executor 不一定真的按顺序读取所有文件。实测中 CC1 跳过了 project-constraints.md，创建了完全不相关的模块。**关键约束和行为准则必须写在 CLAUDE.md 本身**（executor 100% 自动加载），而不是放在 project-constraints.md 中。参考 Karpathy 的 CLAUDE.md 模板（102K stars）：行为准则直接写在 CLAUDE.md 里，不要依赖 "读完 A 再读 B" 的间接指引。
- If the project has no `mini-spec-kit`: check local `/Volumes/ExSSD/projects/mini-spec-kit`; if present, copy it; otherwise clone `https://github.com/zoomc/mini-spec-kit`, then copy it into the actual project root.

### 2.2.1 Context File Prerequisite (CLAUDE.md / AGENTS.md)

不同 agent 读不同的上下文文件：

| Agent | 上下文文件 | 命令/技能格式 |
|-------|-----------|-------------|
| Claude Code | `CLAUDE.md` | `.claude/commands/*.md`（`/specify` 等） |
| Codex | `AGENTS.md` | `.agents/skills/speckit-*/SKILL.md`（`@speckit-specify` 等） |

**Before dispatching:** 验证对应的上下文文件存在且包含 mini-spec-kit workflow 表。如果缺失，先运行 `init-mini-speckit.sh` 或手动创建。

### 2.2.2 New Project Initialization — mini-spec-kit 必须引入

**当任务是创建新项目（而非修改现有项目）时，必须确保 mini-spec-kit 被初始化。初始化由 executor 完成，Hermes 不代劳。**

Hermes 的职责：
- 在确认中明确说明"新项目，需初始化 mini-spec-kit"
- 在 task prompt 中包含初始化指令（见下方模板）
- **不**预先创建目录、git init、copy 文件——这些是 executor 的工作

**Task prompt 模板（新项目）：**

```markdown
读取 CLAUDE.md（或 AGENTS.md），按 Workflow 表执行 mini-spec-kit 流程。

## 初始化（首次执行，项目尚不存在）
1. 创建项目目录并 git init
2. 引入 mini-spec-kit：
   - 优先检查本地 /Volumes/ExSSD/projects/mini-spec-kit，有则 cp -r 到 .mini-spec-kit
   - 否则 git clone https://github.com/zoomc/mini-spec-kit /tmp/mini-spec-kit && cp -r /tmp/mini-spec-kit .mini-spec-kit
3. 确保 CLAUDE.md（或 AGENTS.md）存在且包含 mini-spec-kit workflow 表
4. 完成初始化后，按 Workflow 表执行 Specify 阶段

## 需求
[具体需求描述]

## 模块
[模块名]
```

**Executor 验证清单（executor 自行检查）：**
- [ ] `.mini-spec-kit/` 目录存在且包含 `project-constraints.md`
- [ ] `CLAUDE.md`（Claude Code）或 `AGENTS.md`（Codex）存在且包含 workflow 表
- [ ] `git status` 显示 clean 或仅有上述初始化文件

---

### 2.3 Cost Efficiency
Minimize orchestrator overhead:
- Read only relevant files
- Avoid full-repo scanning
- Avoid frequent log polling
- Delegate heavy work to executors
- Orchestrator only: understand → confirm → dispatch → summarize
- Hermes role must stay minimal: requirement confirmation, one-time executor initial dispatch, final result forwarding, explicit git-only operations, post-executor read-only audits, and explicit deployment operations.
- Hermes must not invent extra agent rounds, self-initiated loops, mid-run tool calls, or mid-run inspections; it must not intervene again due to iterations, call counts, status changes, or any other reason. Overstepping means zero-confidence output.

---

## 3. Execution Modes

### ⚠️ Universal Rule: ALL Tasks Use Dual Session

**Every project code task — no matter how small — MUST use the dual-session file-marker pattern with the full mini-spec-kit 6-phase workflow.** This includes single-character changes, typo fixes, and trivial patches. There are NO exceptions.

- Changing one character in one file → dual session, full 6-phase flow
- Adding one line of code → dual session, full 6-phase flow
- Any task that touches project code → dual session, full 6-phase flow

**Dual session is the default.** It was chosen because:
1. Single session skips spec phases (Specify → Plan → Checklist → Analyze)
2. Single session does not enforce phase gates
3. Single session allows code changes before spec approval
4. These are process violations, not optimizations

The preferred execution pattern is: CC1 (spec+reconcile) + CC2 (implement) with file-marker coordination. See section 3.2 for the pattern.

### 3.2.1 Single-Session MSK Fallback (Recovery Only)

**When dual-session fails repeatedly (CC1 process death, API errors, coordination break), fall back to single-session sequential MSK.** This is a recovery mechanism, not a preferred mode.

**When to use:**
- CC1 dispatch fails 2+ times (process exit, API timeout, auth error)
- File-marker coordination breaks (markers not created, CC2 waits forever)
- API rate limits make dual-session impossible (both sessions hit 429)

#### Pattern A: Full 6-Phase Single Session (preferred if it works)

Run all 6 phases sequentially in one CC session:
```bash
# Single session, all phases
claude --print --max-turns 90 --max-budget-usd 8 \
  --allowedTools 'Read,Edit,Write,Bash' \
  < /tmp/prompt-full-msk.txt
```

**Prompt must include all 6 phases with gate checks:**
```
Execute @speckit-specify, @speckit-plan, @speckit-checklist, @speckit-analyze,
@speckit-implement, @speckit-reconcile in order.

After each phase, run the gate check before proceeding.
Do NOT skip phases. Do NOT modify code before Analyze passes.
```

#### Pattern B: Chunked Single-Session (recovery when Pattern A stalls)

**⚠️ 实测发现（2026-05-10）**：`-p` 模式下单 session 跑全部 6 阶段时，CC 经常在 Specify 后就停止（exit code 0，但只完成 phase 1）。这是因为 `-p` 模式的 agentic loop 在完成一个自认为"完整"的阶段后就退出了。

**解决方案**：按阶段组拆分为多次 dispatch，每次只负责一组阶段：

| Dispatch | 阶段 | Prompt 内容 |
|----------|------|------------|
| 1 | Specify (1) | "读 CLAUDE.md，按 Workflow 表执行 Specify 阶段。创建 spec.md。" |
| 2 | Plan + Checklist + Analyze (2-4) | "读 spec.md，按顺序执行 Plan → Checklist → Analyze。不要修改代码。" |
| 3 | Implement (5) | "读 plan.md，按 TASK 逐个实现。确保编译通过。" |
| 4 | Reconcile (6) | "读 checklist.md，逐项验证并打勾。" |

**每次 dispatch 后必须验证文件落地**：`ls .mini-spec-kit/modules/*/spec.md` 等，确认该阶段产出存在后再 dispatch 下一组。

**适用场景**：小任务（Small/Medium），改动集中，不需要双 session 的审计覆盖。比双 session 更快（无 marker 等待开销），但牺牲了 CC1 reconcile 的独立审计。

**Report to user:** Always disclose that chunked single-session was used. Example: "CC `-p` 模式只完成了 Specify，降级为分段 dispatch（3-4 次串行）完成 MSK 全流程。"

**Validation after single-session (both patterns):**
```bash
# Verify MSK artifacts exist
ls .mini-spec-kit/modules/*/spec.md
ls .mini-spec-kit/modules/*/plan.md
ls .mini-spec-kit/modules/*/checklist.md
# Verify code changes
git diff --stat
```

---

### 3.2 Dual Session File-Marker Pattern (MANDATORY for all tasks)

两个 Claude Code `-p` session 通过文件 marker 协调，完全不需要 tmux：

```
CC1 (Spec + Reconcile)              CC2 (Implement)
terminal(bg, notify=True)           terminal(bg, notify=True)
├── claude -p spec (phase 1-4)      ├── while [ ! -f /tmp/spec-done ]
├── touch /tmp/spec-done            │     sleep 10; done
├── while [ ! -f /tmp/impl-done ]   ├── claude -p implement (phase 5)
│     sleep 10; done                └── touch /tmp/impl-done
└── claude -p reconcile (phase 6)
```

**两个 session 同时 dispatch，通过 marker 文件自动协调，Hermes 零干预。**

CC1 脚本模板：
```bash
#!/bin/bash
# ⚠️ 不要用 set -e！Claude Code 可能返回非零退出码（即使任务成功），
# set -e 会杀掉脚本，导致 touch /tmp/impl-done 永远不执行，协调链断裂。
source ~/.zshrc 2>/dev/null
export ANTHROPIC_BASE_URL ANTHROPIC_AUTH_TOKEN ANTHROPIC_MODEL="${ANTHROPIC_MODEL:-}"
mkdir -p /project
cd /project

# Phase 1-4: Spec (prompt via stdin pipe, NOT shell argument)
claude --print --max-turns 90 --max-budget-usd 5 \
  --allowedTools 'Read,Edit,Write,Bash' \
  < /tmp/prompt-spec.txt

touch /tmp/spec-done

# 等 CC2 实现完成
while [ ! -f /tmp/impl-done ]; do sleep 10; done

# Phase 6: Reconcile
claude --print --max-turns 90 --max-budget-usd 5 \
  --allowedTools 'Read,Edit,Write,Bash' \
  < /tmp/prompt-reconcile.txt
```

**⚠️ Reconcile Prompt 硬约束（防止口头验证）：**

prompt-reconcile.txt 必须包含明确的**动手指令**，不能只说"验证"：

```
## Phase 6: Reconcile — 验证并打勾

1. 读 checklist.md，逐项对照代码实际状态
2. 对每项：执行 Verification 描述的命令/检查
3. 验证通过 → 用 Edit 工具将该项 [ ] 改为 [x]，记录日期
4. 验证失败 → 用 Edit/Write 修复代码，然后重新验证并打勾
5. 全部完成后，执行代码质量检查：
   - 是否有超出 spec 范围的改动？（Surgical Changes）
   - 是否有不必要的抽象/配置/灵活性？（Simplicity First）
   - 是否有未使用的导入/变量/函数？（只清理 implement 阶段引入的）
6. 输出最终 checklist 状态

⚠️ 禁止口头确认：必须实际修改 checklist.md 文件，用 Edit 工具把 [ ] 替换为 [x]。只说"已验证"但不改文件 = 任务失败。
⚠️ 如果发现过度设计，必须简化后再打勾。
```

CC2 脚本模板：
```bash
#!/bin/bash
# ⚠️ 不要用 set -e！touch /tmp/impl-done 必须在脚本最后一行，
# 无论 Claude Code 成功或失败都要执行，否则协调链断裂。
source ~/.zshrc 2>/dev/null
export ANTHROPIC_BASE_URL ANTHROPIC_AUTH_TOKEN ANTHROPIC_MODEL="${ANTHROPIC_MODEL:-}"
mkdir -p /project
cd /project

# 等 CC1 spec 完成
while [ ! -f /tmp/spec-done ]; do sleep 10; done

# Phase 5: Implement (prompt via stdin pipe)
claude --print --max-turns 90 --max-budget-usd 5 \
  --allowedTools 'Read,Edit,Write,Bash' \
  < /tmp/prompt-implement.txt

# ⚠️ 这行必须在最后，无论上面命令是否成功都要执行
touch /tmp/impl-done
```

**⚠️ 关键踩坑（必须遵守）：**
- **⚠️ Prompt 物理隔离（2026-05-01 实测验证）**：每个 prompt 文件只能包含该阶段的指令，**绝对不能**出现其他阶段的内容。否定式约束（"不要做阶段 5-6"）不可靠——LLM 看到 prompt 里有阶段 5 的具体步骤就会执行，不管你怎么说"不要做"。**正确做法：prompt 文件物理上只写该阶段，其他阶段的指令根本不存在于文件中。**
- **prompt 传参**：必须通过 stdin pipe（`< /tmp/prompt-*.txt`），不能用 shell 参数传多行文本——shell 会吞引号/转义导致 `Input must be provided either through stdin or as a prompt argument` 错误
- **环境变量**：必须显式 `export ANTHROPIC_BASE_URL ANTHROPIC_AUTH_TOKEN`，仅 `source ~/.zshrc` 不够——子进程不继承交互式 shell 环境。非 Anthropic provider（如 xiaomimimo）还需 `export ANTHROPIC_MODEL="mimo-v2.5-pro"`——Claude Code 默认发 `claude-sonnet-4-6`，代理会返回 400。详见 §6.1 provider check。。非 Anthropic provider（如 xiaomimimo）还需 `export ANTHROPIC_MODEL="mimo-v2.5-pro"`——Claude Code 默认发 `claude-sonnet-4-6`，代理会返回 400。详见 `claude-code-provider-compat` skill。
- **脚本不要用 `set -e`**：Claude Code `-p` 模式可能返回非零退出码（即使任务成功），`set -e` 会杀掉脚本导致 `touch /tmp/impl-done` 永远不执行，协调链断裂
- **marker 文件**：`touch /tmp/impl-done` 必须是 CC2 脚本最后一行，无论 Claude Code 成功或失败都要执行
- **prompt 文件**：dispatch 前写入 `/tmp/prompt-spec.txt`、`/tmp/prompt-implement.txt`、`/tmp/prompt-reconcile.txt`
- **⚠️ 新项目 `cd` 必须先 `mkdir -p`（2026-05-01 实测验证）**：脚本无 `set -e`，`cd` 到不存在的目录会静默失败，脚本在当前目录（Hermes 工作目录）继续执行，所有文件落在错误位置。**CC1 和 CC2 脚本必须在 `cd` 前加 `mkdir -p /project`**。即使 prompt 里写了"创建项目目录"也没用——executor 还没启动就 `cd` 失败了。

**Hermes dispatch 方式：**
```python
# 同时 dispatch 两个 session
terminal(command="bash /path/to/cc1.sh", background=True, notify_on_complete=True)
terminal(command="bash /path/to/cc2.sh", background=True, notify_on_complete=True)
# 两个都退出后系统自动通知，Hermes 不轮询
```

**关键约束：**
- 两个脚本必须放在项目目录外或 gitignore，避免污染代码
- marker 文件放 `/tmp/` 下，避免残留
- dispatch 前清理旧 marker：`rm -f /tmp/spec-done /tmp/impl-done`
- 每个 session 的 prompt 要简洁，不超过 500 字符

#### Strict no-intervention variant (Codex only)
Use when user explicitly requests Codex. Same file-marker pattern but with Codex as executor:
- Launch A first with `codex exec --full-auto`
- A writes completion marker
- B waits for marker, then runs `codex exec --full-auto`
- Hermes must not poll logs, inspect code, or issue any mid-run follow-up tool calls

**⚠️ Codex 双 session 已验证不可行**：Codex 不按阶段执行，读到需求就一次性做完 spec + implement，不会在 spec 完成后停等 CC2。双 session file-marker 模式仅适用于 Claude Code。如果用户指定 Codex，只能用单 session 一次性 dispatch。

---

### 3.3 Large Tasks
Typical traits:
- Cross-domain (frontend/backend/data/deploy)
- Dirty worktree or multi-stage flow
- Requires multiple validation rounds

Strategy:
- **Must decompose into segments**
- Each segment uses **dual executors independently**
- Never merge multiple domains into one large execution

---

## 4. Dual Executor Rules
- State must be file-based
- Handoff via commit / patch / artifacts
- Reviewer trusts only:
  - repo files
  - diffs
  - logs
  - spec artifacts
- If FAIL / changes requested:
  - loop back to implementer
  - fix only reported issues

Do NOT review a live-changing workspace.

---

## 5. Validation & Deployment

### 5.1 Required Validation
All changes must include minimal relevant validation.

### 5.2 Avoid Over-Validation
- Only validate affected scope
- Skip unrelated checks unless required

### 5.3 Failure Classification
Always distinguish:
- Code failure
- Service failure (e.g. 429)
- Environment/sandbox limits

Do not misclassify environment issues as code bugs.

### 5.3.1 Executor Permission Gaps
If Codex or the execution environment cannot access required local resources (for example Docker socket, launchd-managed services, privileged ports, host-only daemons, host files outside allowed scope):
- classify as `permission` or `env`, not code
- allow fallback validation that does not require the blocked resource (e.g. `docker compose config`, unit tests, build)
- but do **not** claim runtime success for the blocked scope
- final report must explicitly separate:
  - what code/config was fixed
  - what was validated
  - what runtime verification remains blocked by environment permissions

---

### 5.4 Deployment
- Deployment is a separate operational phase
- Do not deploy unless explicitly requested
- Hermes may directly deploy when the user explicitly requests deployment of an already-committed/pushed state
- Deployment may include existing deploy scripts, Docker/compose operations, SSH/cloud CLI deploy commands, and post-deploy health checks
- Hermes must not change code/config or invent a new deployment strategy during deployment
- Must verify branch/commit and working tree status before deployment
- If deployment fails, classify as `code / env / service / permission` and stop; do not patch or redesign without a new confirmed code-gate task

---

## 6. Executor Selection

### Default: Claude Code

**默认使用 Claude Code**，除非用户明确指定 Codex。

选择逻辑：
- 用户说"用 codex" / "走 codex" → 用 Codex
- 其他所有情况 → 用 Claude Code

原因：Claude Code 原生支持 `.claude/commands/` slash 命令，能自动执行 mini-spec-kit 流程；Codex 不会自动调用 `.agents/skills/`，容易跳过 spec 阶段。

### Claude Code 执行规则

1. **必须 source .zshrc** 并传递 env vars（Hermes 子进程不继承交互式 shell 环境变量）
2. 使用 `--print`（非交互批处理模式），不需要 `pty=True`
3. `-C` flag **无效**——用 `workdir` 参数代替
4. **Always use `notify_on_complete=True`**
5. `--max-turns 90`（复杂任务需要足够轮次跑完 speckit 6 阶段）
6. `--max-budget-usd 5`（$0.5 太低，多文件任务容易中断）
7. **⚠️ `-p` 模式静默失败**：`-p` 模式下 Write/Edit/Bash 工具调用可能被静默拒绝，但 Claude 仍会报告完成（exit code 0 + 详细成功文本）。**dispatch 后必须验证文件落地**：`ls .mini-spec-kit/modules/*/spec.md` 和 `git diff --stat`。如果零文件，该 session 产出了幻觉报告。
8. **双 session 模式缓解 `-p` 静默失败**：CC2（implement）静默失败时，CC1（reconcile）会发现代码未落地并报告。实测双 session file-marker 模式 `-p` 写文件成功率远高于单 session。

```python
terminal(
    command="""source ~/.zshrc 2>/dev/null
export ANTHROPIC_BASE_URL="$ANTHROPIC_BASE_URL"
export ANTHROPIC_AUTH_TOKEN="$ANTHROPIC_AUTH_TOKEN"
export ANTHROPIC_MODEL="${ANTHROPIC_MODEL:-}"
claude --print --max-turns 90 --max-budget-usd 5 \
  --allowedTools 'Read,Edit,Write,Bash' \
  < /tmp/prompt-spec.txt""",
    workdir="/absolute/project/path",
    timeout=600,
    notify_on_complete=True
)
# Then wait for the notification — do NOT poll process poll/wait/log
```

### Codex exec syntax note
The current `codex exec` CLI uses `--cd` (not `-C`) and does **not** accept `--allowedTools` as a top-level flag. Use sandbox controls instead, for example:

```bash
codex exec --skip-git-repo-check --sandbox workspace-write --cd /path/to/project < /tmp/prompt.txt
```

If you need a stricter or looser command policy, adjust sandbox settings rather than passing an unsupported tool-allowlist flag. See `references/codex-exec-syntax.md` for the session-tested `codex exec` shape.

### Codex 执行规则（仅用户明确指定时使用）

1. Use: `codex exec --full-auto -C [absolute-project-path] "[task-description]"`
2. Start Codex in background mode with `background=true` and `pty=true`
3. **Always use `notify_on_complete=True`**
4. **Never poll in a loop** — each `process poll/wait/log` = ~10k tokens + 1 iteration
5. Pin Codex to model `gpt-5.4` unless the user explicitly requests a different model
6. `-C` is **mandatory** — must be the project root absolute path
7. Dual-executor tasks must use explicit file boundaries or artifact boundaries
8. **Do NOT use `delegate_task` as a substitute for codex**
9. **⚠️ Codex 不执行 `.agents/skills/`**：Codex 读取 AGENTS.md 文本但永远不会调用 `.agents/skills/speckit-*` slash commands。它会按自己的理解产出类似 spec 的文档，而非遵循实际流程。模块文件可能落在错误位置（如 `./modules/` 而非 `.mini-spec-kit/modules/`）。如果需要严格 speckit 合规，改用 Claude Code。

### Codex Prompt 模板（验证有效）

**关键发现**：Codex 不理解抽象的"按 Workflow 表执行"，但能精确执行写死步骤的指令。Prompt 必须包含：
1. 需求描述（正常写）
2. 每个阶段的具体动作（创建什么文件、包含什么结构）
3. 门禁规则（阶段 X 之前不能做 Y）

```markdown
## 需求
[正常写需求，1-3 行]

## ⚠️ 强制执行规则
你必须按以下 6 个阶段顺序执行，每个阶段完成后才能进入下一个。
**阶段 1-4 完成之前，绝对不能修改任何代码文件（.js/.ejs）。**

## 阶段 1：Specify（写需求规格）
创建文件 `.mini-spec-kit/modules/<module>/spec.md`。
内容必须包含：
- 每个需求一个 REQ-XXX 编号
- 新需求标记 [NEW YYYY-MM-DD]
- 需求描述要具体：输入是什么、输出是什么、边界条件是什么
- 写明 Boundaries（这个模块管什么、不管什么）

## 阶段 2：Plan（写实现计划）
创建文件 `.mini-spec-kit/modules/<module>/plan.md`。
内容必须包含：
- 每个需求对应一个 TASK-XXX
- 每个 TASK 列出：影响哪些文件、具体改什么、步骤是什么
- TASK 和 REQ 的映射关系

## 阶段 3：Checklist（写验证清单）
创建文件 `.mini-spec-kit/modules/<module>/checklist.md`。
内容必须包含：
- 每个 REQ 对应一个 checklist section
- Implementation 子项：描述代码里必须有什么
- Verification 子项：描述怎么验证（具体命令或 URL）
- 所有项初始为 [ ]，不能预勾

## 阶段 4：Analyze（一致性检查）
在 `.mini-spec-kit/modules/<module>/plan.md` 末尾追加 Analysis Report。
检查：
- 每个 REQ 都有对应 TASK
- 每个 TASK 都有对应 checklist 项
- 没有术语漂移
- 结论写 Pass 或 Fail

**只有结论是 Pass 才能进入阶段 5。**

## 阶段 5：Implement（写代码）
现在可以修改代码文件了。按 plan.md 中的 TASK 逐个实现。

## 阶段 6：Reconcile（验证并打勾）
1. 读 checklist.md，逐项验证
2. 验证通过的项把 [ ] 改为 [x]
3. 记录验证日期
4. 如果发现问题，修复后重新验证
```

**不要写**：具体写什么需求内容、具体写什么代码、具体用什么技术方案。那些是 Codex 自己决定的。

```python
terminal(
    command="""codex exec --full-auto -C /path/to/project "读取 AGENTS.md，按 Workflow 表执行 mini-spec-kit 流程。

## 需求
[task description]

## 模块
[module name]" """,
    background=True,
    notify_on_complete=True,
    pty=True
)
# Then wait for the notification — do NOT poll process poll/wait/log
```

**⚠️ Codex 限制**：Codex 不会自动执行 `.agents/skills/` 中的 slash 命令。即使 task prompt 中写了"按 Workflow 表执行"，Codex 可能跳过 spec 阶段直接实现。如果 speckit 流程必须严格执行，优先使用 Claude Code。

### mini-spec-kit 闭环规则（适用于所有执行器，必须写入任务指令）

- 任务指令只需要：(1) 读 CLAUDE.md 或 AGENTS.md (2) 按 Workflow 表执行 (3) 需求描述
- 不要在任务指令中复述 mini-spec-kit 流程细节——上下文文件和命令/skill 文件里已经有了
- 执行器自己会读项目文件获取完整流程定义
- **违反此规则 = 任务指令冗余 + 信息散落在多处 + 新 session 无法复现**

**任务指令模板（简洁版）：**

```markdown
读取 CLAUDE.md（或 AGENTS.md），按 Workflow 表执行 mini-spec-kit 流程。

## 需求
[具体需求描述]

## 模块
[模块名]
```

就这样。不要复述 6 个阶段、不要写流程细节、不要写验证命令。

### Prompt 物理隔离模板（双 session 必须用）

**每个 prompt 文件只包含自己阶段的指令，物理上看不到其他阶段。**

prompt-spec.txt（CC1 用，只含阶段 1-4）：
```
读取 CLAUDE.md，按 Workflow 表执行 mini-spec-kit 流程。

## 初始化（首次执行，项目尚不存在）
1. 创建项目目录并 git init
2. 引入 mini-spec-kit：优先检查本地 /Volumes/ExSSD/projects/mini-spec-kit，有则 cp -r 到 .mini-spec-kit；否则 git clone https://github.com/zoomc/mini-spec-kit /tmp/mini-spec-kit && cp -r /tmp/mini-spec-kit .mini-spec-kit
3. 确保 CLAUDE.md 存在且包含 mini-spec-kit workflow 表
4. 完成初始化后，按 Workflow 表执行 Specify 阶段

## 需求
[具体需求描述]

## 模块
[模块名]

## ⚠️ 你只负责阶段 1-4
你必须按以下 4 个阶段顺序执行：
- 阶段 1：Specify — 创建 .mini-spec-kit/modules/<module>/spec.md
- 阶段 2：Plan — 创建 .mini-spec-kit/modules/<module>/plan.md
- 阶段 3：Checklist — 创建 .mini-spec-kit/modules/<module>/checklist.md
- 阶段 4：Analyze — 在 plan.md 末尾追加 Analysis Report

完成阶段 4 后停止。不要修改任何代码文件（.py/.html/.js/.css）。
不要执行阶段 5 和 6，那是另一个 session 的工作。
```

prompt-implement.txt（CC2 用，只含阶段 5）：
```
## ⚠️ 你只负责阶段 5：Implement

1. 读取 .mini-spec-kit/modules/*/plan.md
2. 按 TASK 逐个实现，修改代码文件（.py/.html/.js/.css）
3. 确保应用可以正常运行

不要执行阶段 1-4（已由另一个 session 完成）。
不要执行阶段 6（由第三个 session 完成）。
```

prompt-reconcile.txt（CC1 用，只含阶段 6）：
```
## ⚠️ 你只负责阶段 6：Reconcile

1. 读 checklist.md，逐项对照代码实际状态
2. 对每项：执行 Verification 描述的命令/检查
3. 验证通过 → 用 Edit 工具将该项 [ ] 改为 [x]，记录日期
4. 验证失败 → 用 Edit/Write 修复代码，然后重新验证并打勾
5. 全部完成后，更新 gate.md：
   - Workflow Status 的 Reconcile: pending → done
   - Current: reconcile → complete
   - Final Result 的 Status: PENDING → PASS
   - History 追加 Reconcile 完成记录
6. 输出最终 checklist 状态

禁止口头确认：必须实际修改 checklist.md 和 gate.md 文件。
如果发现过度设计，必须简化后再打勾。
```

### 6.1 Pre-Dispatch Provider Check (MANDATORY for non-Anthropic providers)

Before dispatching any CC session, verify model compatibility if `ANTHROPIC_BASE_URL` is not `api.anthropic.com`:

```bash
source ~/.zshrc 2>/dev/null
export ANTHROPIC_BASE_URL ANTHROPIC_AUTH_TOKEN ANTHROPIC_MODEL="${ANTHROPIC_MODEL:-}"
claude --print --max-turns 1 --max-budget-usd 1 --allowedTools 'Read' -p "say hi" 2>&1 | head -3
```

- Success → proceed with dispatch
- `400 Not supported model` → load `claude-code-provider-compat` skill, get correct model name
- `429` → rate limit, see §7.1

**⚠️ xiaomimimo 代理模型名陷阱（2026-05-10 实测验证）：** xiaomimimo 代理（`token-plan-cn.xiaomimimo.com/anthropic`）不支持 Claude 原生模型名（如 `claude-sonnet-4-6`、`claude-3-5-sonnet-20241022`）。必须 `export ANTHROPIC_MODEL="mimo-v2.5-pro"`（或 `mimo-v2.5`）。错误表现是 exit code 0 但 stdout 输出 `400 Not supported model`——容易被误判为成功。CC1 和 CC2 会同时失败，浪费 2 个 dispatch cycle。

**Why mandatory**: Both CC1 and CC2 fail in parallel when model is wrong, wasting 2 dispatch cycles. The error has exit code 0 (appears only in stdout), so it's easy to miss.

### Wrong Pattern (DO NOT USE)

---

## 7. Exception Handling
- Dirty worktree → split tasks, avoid large prompts
- Patch mismatch → retry with smaller, current-context task
- Rate limit → treat as service issue, retry smaller task
- Context drift → stop and restart with narrower scope

### 7.1 Executor Fallback: Claude Code 429 → Codex

**When Claude Code hits Anthropic API 429, switch to Codex for implementation.** This is the most common service failure pattern.

**Fallback flow:**
1. Claude Code dispatch fails with 429
2. Create `gate.md` if missing (Codex checks mini-spec-kit gates via AGENTS.md)
3. Dispatch Codex with simplified prompt (skip spec phases, just implement)
4. After Codex completes, dispatch Claude Code for reconcile (may also 429)

**Gate.md bridging (2026-05-07 实测验证):**
Codex reads AGENTS.md which enforces `minispec-gate.sh --phase before-implement`. This gate requires `gate.md` with `Allowed: yes`. If gate.md is missing, Codex stops and asks for confirmation.

```bash
# Create gate.md to pass before-implement gate
cat > .mini-spec-kit/modules/<module>/gate.md << 'EOF'
# <Module> Gate

## Workflow Status
- Specify: done
- Plan: done
- Checklist: done
- Analyze: done
- Implement: pending
- Reconcile: pending

## Gate Status
Current: implement

## Implement Permission
Allowed: yes

## Final Result
Status: PENDING

## Blockers
- None.

## History
- <date>: Implement allowed — fallback from Claude Code 429.
EOF

# Verify gate passes
bash scripts/minispec-gate.sh --phase before-implement --module <module> --project-root .
```

**Codex prompt for implement-only (spec already done):**
```bash
codex exec --full-auto -C /path/to/project "Implement the <module> module. Read .mini-spec-kit/modules/<module>/plan.md and implement TASK-001 through TASK-N. After implementation, run: cd backend && go build ./... && cd ../frontend && npm run build"
```

**Key differences from normal Codex dispatch:**
- Prompt is simpler — just "implement based on plan", no spec phase instructions
- gate.md must exist with `Allowed: yes` before dispatch
- Codex may still hit gate checks for other phases — read its output to identify blockers
- Reconcile after Codex may also 429 — report partial completion if so

---

### 7. Reporting
Do not stream intermediate steps unless needed.
- Never forward platform-generated interrupt/progress/heartbeat notifications to the user.
- For long-running executor work, prefer silent completion mode: one-shot dispatch, no polling loops, no partial logs, no midway status spam.
- Only report intermediate status when the user explicitly asks for updates or when a real blocker/decision is needed.
- If the user prefers final-result-only replies, treat that as a hard communication constraint: no progress pings, no "still running" narration, and no reprinting of platform interrupt messages unless they are needed to explain a blocker.

Report only when:
- task completes
- blocked / risk / decision needed
- user asks
- long task heartbeat

Final report must include:
- status (done / not done)
- execution mode
- changed files/modules
- validation results
- review verdict
- deployment status
- remaining risks/blockers
