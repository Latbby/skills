---
name: agent-workflow
description: 仅当用户明确点名 `agent-workflow` 或手动执行对应命令时使用；不得因一般开发请求自动触发。作为开发全流程的唯一入口，依次调度 agent-ask-me、agent-plan、agent-impl、agent-code-review 和 agent-test，从需求确认持续执行到实现、独立评审与 HTTP 验收，并支持中断后恢复。
---

# Agent Workflow

用一个入口串联现有 skill，不复制或弱化它们的规则。用户显式调用本 skill，即授权按顺序调用下述子 skill；涉及业务选择、权限、生产、破坏性或对外操作时，仍遵守子 skill 的提问和授权边界。

## 输入与状态

接收开发需求，以及可选的 `docs/agent-workflow/<需求标识>/`。未提供目录时从需求生成标识并由 `agent-ask-me` 创建；已有目录时先读取文档状态，从最后一个未完成阶段恢复，不重复已通过阶段，不混用不同 `workflow_id`。

工作流状态以阶段产物为准：

| 阶段 | 调用 | 完成证据 |
|---|---|---|
| 需求 | `agent-ask-me` | `requirements.md` 为 `confirmed` |
| 规划 | `agent-plan` | `spec.md`、`tasks.md`、`test-cases.md` 为 `ready` |
| 实现 | `agent-impl` | 任务全部完成，`reviews.md` 及双轴终审为 `passed` |
| 审计 | `agent-code-review` | 独立评审无 `blocking` 或 `major` 发现 |
| 验收 | `agent-test` | `test-results.md` 总体为 `passed` |

## 调度流程

1. **确认需求**：调用 `agent-ask-me`。只有重大决策需要用户时才暂停；收到回答后沿用原目录继续，直到需求确认。
2. **生成计划**：调用 `agent-plan`，自动产出规格、任务和 HTTP 用例。产物阻塞时停止并报告冲突与恢复条件。
3. **完成实现**：调用 `agent-impl`，逐任务实现、验证、评审和提交，持续到其完成或命中允许阻塞条件。
4. **独立审计**：调用 `agent-code-review` 评审 `base_commit...implementation_head`，同时读取需求、规格和任务。入口调用视为用户已明确要求把结果追加到 `reviews.md` 的“独立审计”章节，但不得改动评审元数据。
5. **闭环修复**：审计存在 `blocking` 或 `major` 时，把发现作为同一需求的修复输入重新调用 `agent-impl`，创建独立修复提交并更新 `implementation_head`，随后再次审计；只保留 `minor` 时记录残余风险并继续。
6. **HTTP 验收**：调用 `agent-test`。验收失败表示工作流未完成：保存证据并重新进入实现、审计、验收闭环；环境或凭据导致的 `blocked` 只报告恢复条件，不擅自访问生产环境。

每次重新进入下游阶段前，校验所有相关文档的 `workflow_id`、状态和提交证据一致。子 skill 的安全限制、验证要求、Git 规范和阻塞条件优先于入口的推进目标。

## 暂停与恢复

只在以下情况暂停：`agent-ask-me` 需要用户重大决策；任一子 skill 命中其阻塞条件；下一步需要新权限或外部状态。暂停时返回当前目录、已完成阶段、阻塞证据、唯一恢复条件和恢复后的下一阶段。

用户再次调用本 skill 并给出同一目录时，从产物状态恢复。不得仅凭对话记忆跳过前置检查，也不得创建第二套工作流绕过阻塞。

## 完成输出

仅当需求、规划、实现、独立审计和 HTTP 验收全部通过时宣布完成。最终汇总：

- 工作流目录和 `workflow_id`；
- 已完成阶段及核心产物；
- 实现提交与评审结论；
- 测试总数、通过数及结果路径；
- 保留的 minor 风险；
- 明确说明未 push。
