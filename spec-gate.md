# Spec Gate

## Purpose

Before any coding work begins, convert the user's informal requirement into a compact, implementation-ready specification.

This skill is a gate before implementation.

Its purpose is to prevent scope drift, premature implementation, unrelated edits, over-engineering, and treating uncertainty as confirmed fact.

After the user confirms the Spec Gate, implementation must be executed according to both of the following:

1. The approved Spec Gate output.
2. All other rules and constraints in this skill.

If implementation requires external technical knowledge, best practices, issue investigation, library behavior confirmation, browser compatibility details, framework-specific behavior, or authoritative references, the Agent may search the web after user confirmation and should prioritize the most official, reliable, and up-to-date sources.

## Aliases

* Spec Gate
* SG
* Requirement Gate
* Scope Gate
* Run Spec Gate
* Gate this first

## Core Rule

When this skill is triggered, do not start implementation.

Do not modify files.
Do not start coding.

The first action must be to rewrite the user's requirement into the fixed `Spec Gate` format and generate a contract document for this task.

Then stop and wait for explicit user confirmation.

Implementation may only begin after the user clearly confirms, for example:

* confirm
* approved
* OK
* go
* start
* proceed
* do it
* implement it
* looks good
* use this

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

The requirement has been received, but the user has not approved it yet.

Before generating the Spec Gate, the Agent should first fully understand the user's requirement.

If the user's requirement is ambiguous, lacks context, involves existing project behavior, depends on project structure, requires confirmation of the current implementation approach, or requires external knowledge to accurately define the requirement scope, the Agent may proactively gather necessary information before writing the Spec Gate, including but not limited to:

* Inspecting project files, code, configuration, documentation, or existing implementation directly related to the requirement.
* Investigating business context and system behavior directly related to the requirement.
* Consulting official documentation, standards, framework documentation, library documentation, browser behavior documentation, security guidance, best practices, and other authoritative technical materials.
* Conducting necessary investigation into the technical approach, compatibility, constraints, or industry conventions involved in the requirement.

Information gathering should be used to accurately understand the requirement, identify constraints, discover potential risks, improve requirement boundaries, and form a higher-quality development plan. It must not be used to start implementation early.

After completing the necessary information gathering, the Agent should output a more accurate and complete Spec Gate based on the obtained information, and clearly mark confirmed facts, inferred content, assumptions, and questions to be confirmed.

Locally create a development task guidance document based on the above information, including requirement understanding, Plan, and Checklist. This document is the mandatory execution document for the development task. During development, after completing each stage, the Agent must return to this document to check, confirm, and update progress.

The Agent must output the requirement summary in the agreed format and generate the corresponding requirement understanding, Plan, and Checklist content for user confirmation.

Before the user confirms, implementation must not begin.

### approved

The user has explicitly confirmed the Spec Gate.

The Agent may now inspect relevant files, plan internally, modify code, run tests, and report results.

The Agent must implement only the approved Spec Gate scope, while continuing to follow all rules in this skill, including anti-drift rules, simplicity rules, verification rules, and done-report rules.

## Principles

Think before coding.

Clarify the target before touching the system.

Prefer a small and accurate spec over a large, fake-complete document.

Do not over-specify details that a modern coding agent can infer from the codebase after approval.

Do not invent APIs, file names, component names, schemas, commands, routes, test names, or implementation details unless the user explicitly mentioned them.

If something is unclear but not blocking, put it under `Assumptions`.

If something could materially change implementation, put it under `Open Questions`.

## Output Format

Always strictly output the following format:

# Spec Gate

## Goal

用 1–3 句话描述本次工作需要达成的目标。

## Scope

列出本次工作必须修改或完成的内容。

## Out of Scope

列出本次工作明确不应修改或涉及的内容。

## Requirement Type

先判断需求类型：

* **Small Change（小需求变更）**：单一功能点、局部逻辑调整、单页面修改、小范围 UI 调整、已有功能优化或缺陷修复。
* **Large Phase（大 Phase / 多模块开发）**：涉及多个功能模块、多个页面、跨系统流程、复杂业务能力建设、阶段性交付目标或需要拆分多个子功能的需求。

根据需求类型选择对应输出粒度：

* Small Change：使用下方的详细行为级描述。
* Large Phase：优先输出功能模块级别内容，聚焦模块目标、模块范围、模块边界、模块间关系及阶段性交付内容，不要求提前细化到每个页面或实现细节。

## Current Behavior

描述当前行为、存在的问题、限制或已知缺口。如未知请填写 `TBD`。

对于 Large Phase，可描述当前业务现状、系统能力缺口或现阶段模块覆盖情况。

## Expected Behavior

描述变更完成后系统应表现出的行为。

对于 Large Phase，可描述整体阶段目标、核心能力建设目标以及各功能模块预期达成的结果。

## Business Rules

列出业务规则、边界情况、校验规则、权限要求、状态变化以及必须保持兼容的约束。

对于 Large Phase，可优先描述模块级业务规则、跨模块约束及关键流程规则。

## Functional Modules

仅在 Large Phase 场景下填写。

列出本阶段涉及的功能模块，并说明：

* 模块目标
* 模块范围
* 模块边界
* 与其他模块的关系
* 本阶段是否包含该模块实现

Small Change 场景填写 `N/A`。

## UI Requirements

描述 PC / Mobile / 响应式行为、文案、状态展示、加载态、空状态、错误状态以及交互要求。

对于 Large Phase，可描述模块级 UI 范围及关键交互要求，无需提前细化所有页面细节。

## Assumptions

列出基于现有信息推断出的假设。如无有意义假设请填写 `None`。

## Open Questions

仅列出会阻塞实现或对实现有重大影响的问题。如无请填写 `None`。

## Gate Checklist

* 目标已明确。
* 范围边界清晰。
* 非范围内容已明确说明。
* 当前行为已记录或标记为 TBD。
* 预期行为已明确说明。
* 业务规则已记录。
* UI 要求已记录。
* 假设条件可见。
* 阻塞性问题已列出。
* 尚未开始实现。

Every Spec Gate response must end with the following:

```md
Please confirm whether this Spec Gate is correct. I will not start implementation until you confirm.
```

## Anti-Drift Rules

Protect the work from these common failures:

* Do not expand scope without permission.
* Do not refactor unrelated code.
* Do not rewrite nearby code just because it looks imperfect.
* Do not change unrelated UI copy, styles, routes, APIs, configuration, tests, or data.
* Do not add new abstractions unless required by the approved scope.
* Do not replace real behavior with mock behavior unless explicitly requested.
* Do not leave temporary bypass, debug, or mock logic in production paths.
* Do not treat assumptions as confirmed facts.
* Do not claim completion without verification.
* Do not start implementation before confirmation.

## Simplicity Rule

The best implementation is the smallest correct change that satisfies the approved spec.

Avoid broad rewrites.

Match the existing code style.

## After Confirmation

After the user confirms, respond briefly:

```md
Confirmed. I will implement only the approved Spec Gate scope and avoid unrelated changes.
```

Then proceed with normal coding-agent work:

1. Inspect only relevant files.
2. Use the approved and locally materialized Spec Gate document as the implementation contract.
3. Follow all rules in this skill during implementation.
4. If necessary for correctness, compatibility, security, or best-practice decisions, investigate authoritative external sources.
5. Make the smallest correct change.
6. Verify the behavior.
7. Report what changed.

The Agent should not treat the Spec Gate output as a loose suggestion. It is the approved implementation boundary.

The Agent should not ignore the other rules in this skill after confirmation. The approved Spec Gate and the full skill rules must remain active until the task is completed.\

## Verification and Delivery Rules

After development is complete, a verification process matching the project type must be performed. The Agent must not claim completion without verification.

Verification requirements are as follows:

1. Prefer running and passing the project's existing automated tests:

   * Unit Test（UT）
   * Integration Test（if the project has it）
   * E2E Test
   * Playwright Test（if the project uses Playwright）

2. The testing strategy should be selected according to the actual project situation:

   * Frontend project: prioritize UT, Playwright, and E2E.
   * Backend project: prioritize UT, Integration Test, and API verification.
   * Full-stack project: run the corresponding tests for all relevant layers.
   * If the project does not have a certain type of test, clearly explain why and do not fabricate test results.

3. If tests fail, defects are found, behavior does not match the requirement, implementation is incomplete, or obvious risks exist:

   * Analyze the cause.
   * Fix the issue.
   * Re-run the relevant tests.
   * Repeat the above process until the approved Spec Gate requirements are met or the blocking reason is clearly stated.

4. The Verification section in the completion report must be based on real execution results. Do not assume tests passed.

5. After implementation, the Agent should perform sufficient multi-round verification and necessary fixes to ensure functionality, edge cases, and related tests have no issues.

   * After completion, delete the requirement document generated in the first step（Spec Gate document）. This document must not be committed to version control, merged, or pushed to the remote repository.
   * Do not keep the requirement document as part of the final code changes.
   * Before merging or pushing, ensure the code is in a releasable state and does not retain temporary debugging, bypass logic, or incomplete implementation.

6. The task may only be closed after all relevant tests pass and the implementation conforms to the approved Spec Gate.

## Done Report

After implementation, report briefly:

# 完成报告

## 摘要

说明本次修改了什么内容。

## 修改文件

列出修改过的文件以及修改原因。

## 验证情况

说明进行了哪些测试或验证。

## 未测试项

说明哪些内容未测试以及原因。

## 剩余风险

列出已知风险或后续需要关注的问题。

## 剩余问题

列出尚未解决的问题、已知限制、技术债务或后续待处理事项。

## 最终 Git 分支状态

说明当前分支名称、提交状态，以及变更是否已提交、已推送、已合并或仍待处理。包含任何相关的分支处理说明或后续需要执行的步骤。

\

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

In the gate phase, unless explicitly requested by the user, do not output detailed implementation plans, API designs, file-change plans, or test commands.

The gate phase should define scope, expected behavior, constraints, assumptions, and blocking questions.

After approval, the Agent may use its normal engineering process to inspect, plan, implement, investigate, test, and report, as long as it always stays within the approved scope and follows this skill.
