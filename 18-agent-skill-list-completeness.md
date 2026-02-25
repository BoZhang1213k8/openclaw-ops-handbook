# 18. Agent 技能列表完整性配置

## 18.1 问题背景

当 Agent 的 `skills/` 目录下存在多个技能时，可能出现以下情况：

1. **新增技能未注册** — 新建了 skill 目录和 SKILL.md，但未在 AGENTS.md 和 TOOLS.md 中登记，Agent 不知道该技能存在
2. **回复依赖上下文记忆** — Agent 回复"可用技能列表"时可能依赖上下文记忆而非实时读取文档，导致：
   - 新增技能后未同步更新文档时遗漏
   - 上下文截断时省略部分技能
   - 无强制规则要求「以文档为准」

**典型表现**：用户问"你能做什么"时，Agent 列出的技能不全，遗漏了已部署但未在文档中注册的技能。

## 18.2 解决方案

### 核心原则

- **TOOLS.md 为唯一权威来源** — Agent 回复技能列表时必须以 TOOLS.md 的「Skills 工具」章节为准
- **三处同步** — 新增技能时需同步更新：AGENTS.md、TOOLS.md、skills/broadband-ops/SKILL.md（若存在元文档）

### 配置要点

| 位置 | 作用 |
|------|------|
| AGENTS.md | 主要职责、标准回复模板中的可用技能、技能列表回复规范 |
| TOOLS.md | Skills 工具完整说明（用途、脚本位置、参数、执行步骤、重要规则） |
| skills/broadband-ops/SKILL.md | 元文档中的「本系统目前支持的功能」列表（可选） |

## 18.3 实施步骤（以 order-archive 为例）

### 步骤 1：在 AGENTS.md 中增加技能列表回复规范

在「状态查询回复规范」的「可用技能列表」说明后增加：

```markdown
- 可用技能列表（仅名称和用途描述，不含路径、脚本名、参数）
  - **回复技能列表时，必须以 TOOLS.md 的「Skills 工具」章节为准，逐项列出，不得遗漏**
  - 若 TOOLS.md 与 skills/ 目录不一致，以 TOOLS.md 为准
```

### 步骤 2：在 AGENTS.md 中补充新技能

- **主要职责**：增加「工单正常归档（通过 order-archive skill）」
- **标准回复模板**：在可用技能中增加「📦 工单正常归档」

### 步骤 3：在 TOOLS.md 中补充完整说明

在「Skills 工具」章节中，按技能顺序新增条目，包含：

- 用途
- 脚本位置
- 使用方法（含示例命令）
- 参数说明表
- 执行步骤（必须按序执行，不得跳过）
- 重要规则（执行前必须用户确认等）

### 步骤 4：更新元文档（可选）

若存在 `skills/broadband-ops/SKILL.md` 等元文档，同步更新「本系统目前支持的功能」列表。

## 18.4 新增技能时的维护流程

每次在 `skills/` 下新增可执行技能时，按以下顺序操作：

```
1. 编写 skill 的 SKILL.md、scripts、skill.json
2. 创建符号链接（若源码在 openClawSkills 中维护）
3. 在 TOOLS.md 的「Skills 工具」中新增完整说明
4. 在 AGENTS.md 的「主要职责」中增加对应条目
5. 在 AGENTS.md 的「标准回复模板」可用技能中增加对应项
6. 在 AGENTS.md 的「常用业务模块」表格中增加对应行
7. 若存在元文档，更新「本系统目前支持的功能」
```

> **完整流程**：broadband-ops-agent 新建 Skill 的端到端标准流程（路径约定、目录结构、符号链接、AGENTS.md/TOOLS.md 更新模板、常见 Skill 类型、检查清单）见 [10.11 broadband-ops-agent 新建 Skill 标准流程](./10-create-skills.md#1011-broadband-ops-agent-新建-skill-标准流程)。

**检查项**：Agent 回复"你能做什么"时，应能逐项列出 TOOLS.md 中登记的所有技能，无遗漏。

## 18.5 与 14 章 Checklist 的衔接

在 [14.9 新 Agent/Skill 上线 Checklist](./14-security-best-practices.md#149-新-agentskill-上线-checklist) 中，Skill 上线已包含「TOOLS.md 已更新」。本配置进一步明确：

- **AGENTS.md 必须同步** — 主要职责 + 标准回复模板
- **技能列表回复规范** — 强制以 TOOLS.md 为准，避免依赖上下文记忆

建议在 Skill 上线 Checklist 中增加：

```
- [ ] AGENTS.md 主要职责和标准回复模板已同步更新
- [ ] 技能列表回复规范已配置（以 TOOLS.md 为准）
```
