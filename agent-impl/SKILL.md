---
name: agent-impl
description: 仅当用户明确点名 `agent-impl` 或手动执行对应命令时使用；不得因实现请求或已有任务清单自动触发。按 agent-plan 产物逐项实现、验证、双轴初审并创建中文原子提交；复杂任务使用子代理，最终由两个独立子代理全面复审，除明确阻塞外持续到全部完成。
---

# Agent Impl

接收 `docs/agent-workflow/<需求标识>/`，一次完成一个任务并提交，不累计问题到最后。

## 初始化与恢复

1. 校验 `requirements.md` 为 `confirmed`，`spec.md`、`test-cases.md` 为 `ready`，四份输入的 `workflow_id` 均与目录一致；`tasks.md` 首次为 `ready`，恢复时可为 `in_progress | blocked`。
2. 读取适用规范和构建配置，确认 Git `HEAD`；首次记为 `base_commit`，恢复时校验已有记录。
3. 保护并排除无关用户改动；冲突无法隔离时阻塞。使用现有 Git 身份，禁止 push 和破坏性命令。
4. 首次创建、恢复时更新 `reviews.md`，并把 `tasks.md` 改为 `stage: impl`、`status: in_progress`：

```yaml
---
workflow_id: <需求标识>
stage: impl
status: in_progress # in_progress | passed | blocked
base_commit: <提交>
implementation_head: pending
final_code_review: pending # pending | passed | blocked
final_requirements_review: pending
---
```

恢复时只允许一个 `in_progress` 任务；结合工作区、提交和评审记录续做。重新检查 `blocked` 的恢复条件。状态矛盾或多个进行中任务时阻塞。任务已全部完成但终审阻塞时，直接恢复终审。

由 `agent-workflow` 因独立审计的 `blocking/major` 发现或 HTTP 验收失败重新调用时，进入返修模式：确认问题属于原需求范围，把 `reviews.md` 改回 `in_progress`，逐项记录证据并完成最小修复、相关验证、双轴复审和独立中文修复提交，更新 `implementation_head` 后恢复三个评审字段为 `passed`。不改写已完成任务；超出原需求的发现按阻塞处理，交回需求阶段决定范围。

## 逐任务循环

按依赖选择任务并标为 `in_progress`：

1. 仅修改任务所需组件，范围外问题只记入 `reviews.md`。
2. `complex` 任务必须拆成边界清晰、不重叠的子任务交给子代理；主代理整合并负责。环境无子代理时阻塞。
3. 最小化实现，复用已有代码并遵循仓库风格；方法注释说明作用、入参、出参，关键逻辑添加单行注释。
4. 运行任务指定的测试、构建、类型或静态检查，使用真实业务数据。失败时修复根因，不绕过检查。
5. 主代理分别进行代码轴（正确性、异常、安全、维护性、规范、范围、测试）和需求轴（需求、规格、任务、遗漏、错误或额外行为）初审。
6. 将发现按 `blocking | major | minor` 写入评审记录；修复全部 blocking/major 后重验、重审，保留 minor 时说明理由。
7. 每次验证记录目的、入参、出参、预期和实际结果。
8. 通过后将任务改为 `completed`；最后一项同时令任务文档 `status: completed`。更新评审记录，检查并仅暂存本任务代码、测试和工作流文档，排除凭据与无关文件，创建中文原子提交。

## 最终复审

全部任务完成后，把当前提交记为 `implementation_head`，固定评审范围为 `base_commit...implementation_head`，同时启动两个互不共享结论的子代理：代码评审只检查代码轴；需求评审读取需求、规格和任务，只检查需求轴。环境不能提供两个独立子代理时阻塞。

将带文件证据的分级发现并列写入 `reviews.md`。存在 blocking/major 时修复、验证、创建独立中文提交、更新实现头并重复双审。通过后运行与范围相称的完整验证并记录五要素，将 `status`、`final_code_review`、`final_requirements_review` 设为 `passed`，再创建仅含工作流记录的中文文档提交；该记录提交不进入实现评审范围。

## 允许阻塞

仅限：缺少权限/凭据/外部服务；需要未授权的破坏性、生产或对外操作；确认需求硬冲突；同一外部环境失败连续三次且无替代验证；用户改动冲突无法隔离；复杂任务无子代理；最终无法取得两个独立子代理。

阻塞时标记当前任务（如已进入）、`tasks.md` 和 `reviews.md`，记录证据、尝试和唯一恢复条件。只有所有任务完成且有提交与初审、双轴终审和最终验证通过、技能产生的改动均已提交、从未 push，才宣布完成。
