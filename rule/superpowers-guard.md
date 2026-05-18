---
alwaysApply: false
description: 当执行 superpowers-automation-guide 流程时，遵循本规则
---

## 核心规则

> 以下规则确保 Superpowers 自动化流程（brainstorming / prd-driven-development → writing-plans → subagent-driven-development → finishing）始终处于正常、可控的状态。AI 在执行任何阶段前**必须**遵守这些约束。

### G0: 服务端优先约束（最高优先级）

本流程中所有阶段**只着重考虑服务端代码**，客户端不在实现范围内。核心目标：**确保服务端代码提供完整、正确、可对接的 API 接口，使客户端能够顺利调用和集成**。

| 阶段 | 约束 |
|------|------|
| **brainstorming / PRD 分析** | 聚焦服务端架构、数据模型、API 契约（接口路径、请求/响应结构、错误码），不需要设计客户端 UI 或交互 |
| **writing-plans** | Task 拆分只包含服务端实现任务（Facade、Service、DAO、DTO 等），无需规划客户端开发任务 |
| **subagent-driven-development** | 子代理只编写服务端代码，确保接口定义清晰、文档完整，方便客户端对接 |
| **审查（Spec Review / Code Quality Review）** | 重点审查接口契约的完整性和一致性，确保不会遗漏客户端所需的字段或能力 |

- **禁止**在设计或计划中引入客户端（前端/移动端）的实现任务
- **必须**确保每个 API 接口都有明确的请求/响应结构定义，使客户端可以直接对接

### G1: 前置环境检查

在启动任何入口路径之前，**必须**完成以下检查，任一项不通过则阻塞并提示用户：

| 检查项 | 验证方式 | 不通过时的处理 |
|-------|---------|--------------|
| **Git 工作区干净** | `git status` 无未提交的变更 | 提示用户先 commit 或 stash，不得继续 |
| **当前分支正确** | 确认在预期的基线分支上（如 `main`/`master`/`develop`） | 提示用户切换分支或确认当前分支可用 |
| **项目可编译** | 执行项目构建命令（如 `mvn compile`）确认无编译错误 | 提示用户先修复编译错误，不得继续 |
| **计划/设计文档存在**（从中间阶段切入时） | 检查用户指定的文件路径是否存在且内容有效 | 提示文件不存在或内容为空，引导用户回到上游阶段 |

### G2: 阶段门禁（不可跳跃）

每个阶段的产出是下一阶段的**强制输入**，禁止跳过：

```
brainstorming / prd-driven-development
    │
    │  ✅ 必须产出：设计文档（.aone_copilot/docs/superpowers/specs/）
    │  ⛔ 无设计文档 → 不得进入 writing-plans
    ▼
writing-plans
    │
    │  ✅ 必须产出：实施计划（.aone_copilot/docs/superpowers/plans/）
    │  ⛔ 无实施计划 → 不得进入 subagent-driven-development
    ▼
subagent-driven-development
    │
    │  ✅ 必须产出：全部 Task 通过双阶段审查
    │  ⛔ 有 Task 未通过审查 → 不得进入 finishing
    ▼
finishing-a-development-branch
```

- **唯一例外**：用户明确输入 `/skip-to <阶段名>` 可强制跳转，但 AI 必须发出警告并记录跳过原因。

### G3: 检查点强制确认

每个阶段完成后，**必须**输出结构化状态摘要，并等待用户确认后才能继续：

```
✅ [阶段名] 已完成
  - 产出物：[列出具体文件路径]
  - 关键决策：[列出本阶段的重要决策]

⏳ 等待确认：是否进入下一阶段 [下一阶段名]？
  - [continue] 继续
  - [retry] 重做当前阶段
  - [abort] 终止整个流程
```

- AI **禁止**在未收到用户 `continue` 确认前自动进入下一阶段。
- 用户输入 `abort` 时，必须立即停止，不得继续执行任何后续操作。

### G4: 子代理执行守护

在 `subagent-driven-development` 的 Task 循环中，强制遵守以下规则：

| 规则 | 说明 |
|------|------|
| **单 Task 串行** | 同一时间只能有一个 Task 处于执行状态，前一个 Task 必须通过双阶段审查后才能开始下一个 |
| **审查不通过 ≤ 3 次重试** | 同一 Task 的 Spec Review 或 Code Quality Review 不通过时，最多重试 3 次。超过 3 次必须暂停并请求用户介入 |
| **编译/测试必须通过** | 每个 Task 的 Implementer 子代理完成后，必须确认编译和测试全部通过，不得跳过 |
| **不得修改非本 Task 范围的代码** | 子代理只能修改当前 Task 计划中指定的文件和模块，禁止越界修改 |

### G5: 异常处理与回滚

| 异常场景 | 处理方式 |
|---------|---------|
| **子代理执行失败**（编译错误/测试失败） | 暂停流程 → 输出错误详情 → 尝试自动修复（最多 3 次） → 仍失败则请求用户介入 |
| **Git worktree 创建失败** | 检查磁盘空间和 Git 状态 → 提示用户手动修复 → 不得在主工作区直接执行 |
| **网络/工具调用超时** | 自动重试 1 次 → 仍失败则暂停并告知用户 |
| **用户长时间未响应检查点** | 保持等待状态，不得自动跳过或继续 |
| **流程中途 abort** | 保留已完成的产出物 → 清理未完成的临时文件 → 输出当前进度摘要，方便后续恢复 |

### G6: 产出物完整性验证

在进入 `finishing-a-development-branch` 之前，**必须**验证：

- [ ] 所有 Task 在计划文件中已标记为 ✅ 完成
- [ ] 所有测试通过（`mvn test` 或对应测试命令）
- [ ] 无遗留的 lint 错误
- [ ] 设计文档和实施计划文件与最终代码一致（如有阶段性调整，文档已同步更新）
- [ ] Git 提交历史清晰，每个 Task 有独立的、有意义的 commit message

任一项不通过，**禁止**进入 finishing 阶段。

### G7: 流程状态追踪

整个流程执行期间，AI 必须维护一个**流程状态表**，实时更新并在用户询问时展示：

```
📊 Superpowers 流程状态
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  阶段                      状态        产出物
  brainstorming / PRD      ✅ 完成     specs/2026-04-24-xxx.md
  writing-plans            ✅ 完成     plans/2026-04-24-xxx.md
  git-worktree             ✅ 完成     /path/to/worktree
  subagent Task 1/5        ✅ 通过     commit abc1234
  subagent Task 2/5        🔄 执行中   —
  subagent Task 3/5        ⏳ 待执行   —
  subagent Task 4/5        ⏳ 待执行   —
  subagent Task 5/5        ⏳ 待执行   —
  finishing                ⏳ 待执行   —
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```
