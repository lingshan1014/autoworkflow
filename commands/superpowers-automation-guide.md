# Superpowers 自动化执行指南

> 本文档总结了 Superpowers 插件的完整 Skill 调用链路和自动化执行方式，帮助你用最少的人工介入完成从想法到代码的全流程。

---

## 完整 Skill 调用链路

```
你只需说一句话
       │
       ▼
  ┌─────────────────────────────────────────────────┐
  │ 入口 skill（手动触发，二选一）                      │
  │                                                  │
  │  "使用 brainstorming 技能，我想做 X"               │
  │  "使用 prd-driven-development 技能"               │
  └──────────────────────┬──────────────────────────┘
                         │ 自动
                         ▼
              writing-plans
              （生成分步实施计划）
                         │ 你选 "Subagent-Driven"
                         ▼
              using-git-worktrees  ← 自动
              （创建隔离工作区）
                         │ 自动
                         ▼
              subagent-driven-development  ← 自动
              ┌──────────────────────────────────┐
              │  每个 Task 自动循环：              │
              │  ① implementer（遵循 TDD skill）  │
              │  ② spec-reviewer（规格审查）       │
              │  ③ code-quality-reviewer（质量审查）│
              │  ④ 通过 → 下一个 Task             │
              └──────────────────────────────────┘
                         │ 全部完成，自动
                         ▼
              finishing-a-development-branch
              （你选择：合并 / PR / 保留 / 丢弃）
                         │
                         ▼
                       完成 ✅
```

---

## 两条入口路径

### 路径 A：从想法到代码（无 PRD）

**第 1 步 — 启动头脑风暴（手动触发）：**

```
使用 brainstorming 技能，我想做 [描述你的想法]
```

- AI 会逐个提问、提出 2-3 个方案、分段呈现设计
- 你只需回答问题 + 确认设计
- 设计通过后，AI **自动调用** `writing-plans`

**第 2 步 — 生成实施计划（自动衔接）：**

- `brainstorming` 结束后自动调用 `writing-plans`
- 生成分步实施计划，保存到 `docs/superpowers/plans/`
- 计划完成后 AI 会问你选择执行方式
- 你回答：**Subagent-Driven**（推荐）

**第 3 步 — 自动执行（自动衔接）：**

- 自动创建 git worktree（调用 `using-git-worktrees`）
- 逐个 Task 派子代理：实现 → 规格审查 → 质量审查
- 每个 Task 内部遵循 `test-driven-development`
- 全部完成后自动调用 `finishing-a-development-branch`
- 你选择：合并 / PR / 保留 / 丢弃

---

### 路径 B：从 PRD 文档到代码（有文档）

**第 1 步 — 启动 PRD 分析（手动触发）：**

```
使用 prd-driven-development 技能，PRD 文档链接：[你的链接]
```

- AI 读取 PRD → 总结需求 → 逐个澄清歧义 → 分析代码库 → 生成技术方案
- 你确认技术方案
- AI **自动调用** `writing-plans`

**第 2-3 步 — 与路径 A 完全相同（自动衔接）：**

- `writing-plans` → 选择 Subagent-Driven → `subagent-driven-development` → `finishing`

---

## 从任意阶段切入

如果你已经有了中间产物，可以跳过前面的步骤，直接从对应阶段开始：

| 你手头有什么 | 启动命令 |
|------------|---------|
| **只有想法** | `使用 brainstorming 技能，我想做 X` |
| **有 PRD 文档** | `使用 prd-driven-development 技能，PRD 链接：[URL]` |
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
| `test-driven-development` | 所有实现代码必须遵循 TDD（红-绿-重构） |
| `requesting-code-review` | 每个 Task 完成后请求代码审查 |
| `receiving-code-review` | 收到审查反馈后的处理规范 |
| `verification-before-completion` | 声称完成前必须运行验证 |
| `systematic-debugging` | 遇到 bug 时的系统化调试流程 |
| `dispatching-parallel-agents` | 有 2+ 独立任务时并行派发子代理 |

---

## Subagent-Driven 执行的内部循环

每个 Task 的执行过程（全自动）：

```
┌─────────────────────────────────────────────────┐
│                  单个 Task 循环                   │
│                                                  │
│  ① Implementer 子代理                            │
│     - 遵循 test-driven-development               │
│     - 写测试 → 看失败 → 写代码 → 看通过 → 提交     │
│     - 自我审查                                    │
│              │                                   │
│              ▼                                   │
│  ② Spec Reviewer 子代理                          │
│     - 检查代码是否符合规格（不多不少）               │
│     - ❌ 不通过 → 返回 ① 修复 → 重新审查           │
│     - ✅ 通过 → 继续                              │
│              │                                   │
│              ▼                                   │
│  ③ Code Quality Reviewer 子代理                  │
│     - 检查代码质量                                │
│     - ❌ 不通过 → 返回 ① 修复 → 重新审查           │
│     - ✅ 通过 → 标记 Task 完成                    │
│              │                                   │
│              ▼                                   │
│  ④ 下一个 Task（或全部完成 → finishing）           │
└─────────────────────────────────────────────────┘
```

---

## 快速启动模板（复制即用）

**场景 1 — 有想法：**

```
使用 brainstorming 技能。
我想给项目添加一个用户认证模块，支持 JWT 和 OAuth2。
```

**场景 2 — 有 PRD：**

```
使用 prd-driven-development 技能。
PRD 文档链接：https://alidocs.dingtalk.com/i/nodes/xxxxx
```

**场景 3 — 已有设计文档，直接写计划：**

```
使用 writing-plans 技能。
设计文档在 docs/superpowers/specs/2026-04-15-auth-design.md
```

**场景 4 — 已有计划，直接执行：**

```
使用 subagent-driven-development 技能。
计划文件在 docs/superpowers/plans/2026-04-15-auth-plan.md
```
