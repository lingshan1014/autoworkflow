# Superpowers 自动化执行指南

> **🎯 核心约束：服务端代码优先 + 角色自动识别**
>
> 本流程中所有阶段（brainstorming / PRD 分析 / 设计 / 实施计划 / 编码）**只需着重考虑服务端代码**，不涉及前端/移动端 UI 或交互的实现。
>
> **⚠️ 角色识别规则**：在开始设计和编码前，必须先判断本项目在当前需求中的角色：
>
> **场景 A — 作为服务端提供接口（默认）**：当需求是"实现某个功能/业务"、"提供 API 给别人调用"时，按此模式执行。
> - **设计阶段**：聚焦服务端架构、数据模型、API 契约（接口路径、请求/响应结构、错误码），不需要设计客户端 UI 或交互
> - **计划阶段**：Task 拆分只包含服务端的实现任务，无需规划客户端开发任务
> - **编码阶段**：只编写服务端代码（Facade、Service、DAO、DTO 等），确保接口定义清晰、文档完整，方便客户端对接
> - **审查阶段**：重点审查接口契约的完整性和一致性，确保不会遗漏客户端所需的字段或能力
>
> **场景 B — 作为客户端对接外部 API**：当用户提供了外部 API 文档/接口规范，且意图是"调用/对接/集成这些接口"时，按此模式执行。
> - **设计阶段**：聚焦如何在本项目的服务端代码中对接外部 API——设计 Integration 层的客户端调用逻辑、请求/响应的 DTO 映射、错误处理与重试策略、防腐层设计
> - **计划阶段**：Task 拆分围绕集成层实现，包括：外部 API 客户端封装、DTO 定义与转换、调用逻辑编写、异常处理、必要的本地业务适配
> - **编码阶段**：编写 Integration 模块的客户端代码（HTTP Client 封装、请求构建、响应解析、错误映射等），**外部 API 文档中的接口是要被调用的，不是要被实现的**
> - **审查阶段**：重点审查外部 API 的调用是否正确（路径、参数、鉴权）、错误处理是否完善、防腐层是否有效隔离了外部模型变化
>
> **如何判断**：如果用户提供了 API 文档并使用了"对接"、"调用"、"集成"、"接入"等表述，或文档中的接口明显不属于本项目（如第三方平台、上游系统），则按**场景 B** 执行。如不确定，主动询问用户。

## 完整 Skill 调用链路

1. **入口 skill**（手动触发，二选一）
    - "使用 brainstorming 技能，我想做 X"
    - "使用 prd-driven-development 技能"

2. **writing-plans**（自动触发）→ 生成分步实施计划
    - 用户选择执行方式："Subagent-Driven"

3. **using-git-worktrees**（自动触发）→ 创建隔离工作区

4. **subagent-driven-development**（自动触发）→ 每个 Task 循环执行以下步骤：
    - 4.1 **code-guidelines** — 编码前思考
    - 4.2 **implementer** — 遵循 TDD skill 实现代码
    - 4.3 **spec-reviewer** — 规格审查（不通过则回到 4.2 修复）
    - 4.4 **code-quality-reviewer** — 质量审查（不通过则回到 4.2 修复）
    - 4.5 通过 → 进入下一个 Task，直到全部完成

5. **finishing-a-development-branch**（全部完成后自动触发）
    - 用户选择：合并 / PR / 保留 / 丢弃

6. **完成 ✅**

---

## 两条入口路径

### 路径 A：从想法到代码（无 PRD）

**第 1 步 — 启动头脑风暴（手动触发）：**

```
使用 brainstorming 技能，我想做 [描述你的想法]
```

- AI 与你交互确认设计方案

**第 2 步 — 生成实施计划（自动衔接）：**

- `brainstorming` 结束后自动调用 `writing-plans`
- 生成分步实施计划，保存到 `.aone_copilot/docs/superpowers/specs/`
- 计划完成后 AI 会问你选择执行方式
- 你回答：**Subagent-Driven**（推荐）

**第 3 步 — 自动执行（自动衔接）：**

- 自动创建 git worktree（调用 `using-git-worktrees`），分支名必须遵循格式：`<类型>/<日期>_<描述>`
    - **类型**：`feature` / `bugfix` / `hotfix` / `refactor` 等
    - **日期**：`YYYYMMDD` 格式，取创建当天
    - **描述**：小写英文，单词间用 `-` 连接，简要概括功能
    - 示例：`feature/20260427_add-login-api`、`bugfix/20260427_fix-order-query`
- 逐个 Task 派子代理：编码前思考 → 实现 → 规格审查 → 质量审查
- 每个 Task 开始前先遵循 `code-guidelines` 进行思考，再遵循 `test-driven-development` 实现
- 全部完成后自动调用 `finishing-a-development-branch`
- 你选择：合并 / PR / 保留 / 丢弃

---

### 路径 B：从 PRD 文档到代码（有文档）

**第 1 步 — 启动 PRD 分析（手动触发）：**

```
使用 prd-driven-development 技能，PRD 文档：[文档路径或链接]
```

- AI 分析 PRD 并生成技术方案

**第 2-3 步 — 与路径 A 完全相同（自动衔接）：**

- `writing-plans` → 选择 Subagent-Driven → `subagent-driven-development` → `finishing`

---

## 从任意阶段切入

如果你已经有了中间产物，可以跳过前面的步骤，直接从对应阶段开始：

| 你手头有什么 | 启动命令 |
|------------|---------|
| **只有想法** | `使用 brainstorming 技能，我想做 X` |
| **有 PRD 文档** | `使用 prd-driven-development 技能，PRD 文档：[文档路径或链接]` |
| **有设计文档** | `使用 writing-plans 技能，设计文档在 docs/superpowers/specs/YYYY-MM-DD-xxx.md` |
| **有实施计划** | `使用 subagent-driven-development 技能，计划文件在 docs/superpowers/plans/YYYY-MM-DD-xxx.md` |

---

## 人工介入点清单

整个流程中，**最少只需人工介入 3 次**：

| 序号 | 何时 | 你需要做什么 | 预计耗时 |
|------|------|-------------|---------|
| 1 | brainstorming / PRD 分析阶段 | 回答问题 + 确认设计/方案 | 5-15 分钟 |
| 2 | writing-plans 完成后 | 说一句 "Subagent-Driven" | 3 秒 |
| 3 | 所有代码完成后 | 选择合并方式（1/2/3/4） | 3 秒 |

> 中间的编码、测试、代码审查、修复全部自动完成，无需人工干预。

---

## 各 Skill 职责速查

### 核心流程 Skill

| Skill | 职责 | 产出物 |
|-------|------|--------|
| `brainstorming` | 探索想法 → 提问 → 提出方案 → 写设计文档 | `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md` |
| `prd-driven-development` | 读取 PRD → 提取需求 → 澄清歧义 → 生成技术方案 | `docs/superpowers/plans/YYYY-MM-DD-<feature>.md` |
| `writing-plans` | 将设计/方案拆解为极细粒度的分步实施计划 | `docs/superpowers/plans/YYYY-MM-DD-<feature>.md` |
| `using-git-worktrees` | 创建隔离的 git worktree 工作空间 | 独立的工作目录 + 干净的基线测试 |
| `subagent-driven-development` | 每个 Task 派子代理实现 + 两阶段审查 | 已实现、已测试、已审查的代码 |
| `executing-plans` | 在当前会话内联执行计划（无子代理时的备选） | 已实现的代码 |
| `finishing-a-development-branch` | 验证测试 → 提供 4 个完成选项 | 合并/PR/保留/丢弃 |

### 辅助 Skill

| Skill | 职责 |
|-------|------|
| `code-guidelines` | 编码前思考 — 明确假设、最简方案、手术式修改、目标驱动执行 |
| `test-driven-development` | 所有实现代码必须遵循 TDD（红-绿-重构） |
| `requesting-code-review` | 每个 Task 完成后请求代码审查 |
| `receiving-code-review` | 收到审查反馈后的处理规范 |
| `verification-before-completion` | 声称完成前必须运行验证 |
| `systematic-debugging` | 遇到 bug 时的系统化调试流程 |
| `dispatching-parallel-agents` | 有 2+ 独立任务时并行派发子代理 |

---