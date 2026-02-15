# 2. 核心概念

## 2.1 架构概览

```
┌─────────────────────────────────────────────────────────────┐
│                    OpenClaw 网关 (Gateway)                    │
│                     端口: 18789                               │
├──────────┬──────────┬──────────────────────────────────────────┤
│          │          │                                          │
│ Telegram │   飞书   │            HTTP API                      │
│  Plugin  │  Plugin  │         (chatCompletions)                │
│          │          │                                          │
├──────────┴──────────┴──────────────────────────────────────────┤
│                                                               │
│                    Bindings（渠道路由）                         │
│           渠道 + 群组/对话 ID → 指定 Agent                      │
│                                                               │
├──────────┬──────────┬──────────┬──────────────────────────────┤
│          │          │          │                               │
│  Agent   │  Agent   │  Agent   │  Agent N ...                 │
│ Manager  │   Ops    │ Broadband│                               │
│          │          │   Ops    │                               │
├──────────┼──────────┼──────────┼──────────────────────────────┤
│          │          │          │                               │
│  Skills  │  Skills  │  Skills  │   Skills                      │
│          │          │          │                               │
└──────────┴──────────┴──────────┴──────────────────────────────┘
```

## 2.2 核心概念

### Agent（智能体）

Agent 是 OpenClaw 中的核心工作单元。每个 Agent：

- 有唯一的 `id`（如 `ops`、`broadband-ops`）
- 绑定一个 AI 模型
- 拥有独立的 workspace（工作区）
- 通过 Skills 获得专业能力
- 可以绑定到通信渠道（Telegram/飞书群组）

### Workspace（工作区）

每个 Agent 拥有独立的工作区，位于 `~/.openclaw/workspace/<workspace-name>/`。

> **注意：** workspace 目录名不一定等于 Agent ID。例如 Agent ID 为 `ops`，workspace 目录名可能是 `ops-agent`；Agent ID 为 `broadband-ops`，workspace 目录名可能是 `broadband-ops-agent`。在 `openclaw.json` 的 `agents.list` 中通过 `workspace` 字段指定完整路径。

工作区包含：

```
workspace/<workspace-name>/
├── AGENTS.md      # 工作职责、安全规则（Agent 的"宪法"）
├── SOUL.md        # 核心特质、沟通风格
├── IDENTITY.md    # 身份信息（名称、Emoji）
├── TOOLS.md       # 可用工具和技能说明
├── BOOTSTRAP.md   # 启动配置
├── USER.md        # 服务对象信息
├── HEARTBEAT.md   # 心跳配置
├── MEMORY.md      # 长期记忆
├── memory/        # 每日记忆
│   └── YYYY-MM-DD.md
└── skills/        # Agent 专属 Skills
    └── <skill-name>/
```

#### 关键文件说明

| 文件 | 作用 | 优先级 |
|------|------|--------|
| `AGENTS.md` | 定义工作范围、安全规则、Prompt 注入防护 | 最高 |
| `SOUL.md` | 定义人格特质、沟通风格、工作原则 | 高 |
| `IDENTITY.md` | 身份标识（Name、Emoji） | 中 |
| `TOOLS.md` | 可用工具清单和使用说明 | 中 |
| `BOOTSTRAP.md` | 首次启动指令 | 仅启动时 |
| `MEMORY.md` | 长期知识库 | 持续参考 |

### Skill（技能）

Skill 赋予 Agent 执行具体任务的能力：

```
skills/<skill-name>/
├── skill.json     # 技能元数据、命令定义
├── SKILL.md       # Agent 使用指南、安全规则
└── scripts/       # 执行脚本
    ├── run.sh
    ├── main.py
    └── .env       # 环境变量
```

### Binding（绑定）

Binding 将通信渠道的消息路由到指定 Agent：

```
渠道（Telegram/飞书） + 群组 ID → Agent ID
```

### Policy（策略）

Policy 控制 Agent 的权限边界：

- **工具权限**：哪些 Agent 可以用哪些工具
- **文件权限**：哪些 Agent 可以读写哪些文件
- **技能权限**：哪些 Agent 可以使用哪些 Skill

## 2.3 Agent 间的隔离原则

```
┌──────────────────────────────────────────┐
│              Agent Manager               │
│  (调度中枢，不直接执行业务操作)            │
│  可用工具: agents_list, sessions_send     │
├──────────────────────────────────────────┤
│                   ↕                      │
├──────────────────┬───────────────────────┤
│     OpsBot       │  BroadbandOpsBot      │
│     运维监控      │  家宽运维              │
│                  │                       │
│ ❌ 不可访问 BroadbandOpsBot 的 workspace   │
│ ❌ 不可访问其他 Agent 的 skills             │
│ ❌ 不可访问 openclaw.json 和 policy.yaml   │
└──────────────────┴───────────────────────┘
```

**核心原则：**
1. 每个 Agent 只能访问自己的 workspace
2. 每个 Agent 只能使用分配给它的 Skills
3. 系统配置文件对所有 Agent 不可见
4. Agent Manager 负责跨 Agent 协调，但不能访问其他 Agent 的私有数据
