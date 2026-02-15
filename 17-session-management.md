# 17. 会话管理与性能优化

## 17.1 会话生命周期

OpenClaw Gateway 为每个 Agent + 渠道组合维护独立的会话。

```
用户消息 → Gateway → 查找/创建会话 → 发送给 LLM → 回复 → 更新会话
```

### 会话类型

| 类型 | 渠道标识 | 生命周期 | 说明 |
|------|----------|---------|------|
| 渠道持久会话 | `telegram:group:*` / `feishu:group:*` / `dingtalk:*` / `main` | 长期存在 | 与群组/控制台绑定，持续积累上下文 |
| 临时会话 | `openai:*` / `cron:*` | 按需创建 | HTTP API / Web 聊天 / 定时任务产生 |

### 会话存储结构

每个 Agent 的会话数据存储在：

```
~/.openclaw/agents/<agent-id>/sessions/
├── sessions.json              # 会话索引（所有会话的元数据）
├── <session-id-1>.jsonl       # 会话历史（消息记录）
├── <session-id-2>.jsonl
└── ...
```

`sessions.json` 中每条记录包含关键字段：

| 字段 | 说明 |
|------|------|
| `sessionId` | 会话唯一标识 |
| `updatedAt` | 最后活跃时间戳 (ms) |
| `totalTokens` | 当前已使用的 token 数 |
| `contextTokens` | 模型的上下文窗口大小（如 256,000） |
| `inputTokens` | 累计输入 token 数 |
| `compactionCount` | compaction 触发次数 |

## 17.2 Compaction 机制

### 什么是 Compaction

当会话积累的上下文接近模型的最大窗口时，OpenClaw 应通过 compaction 将旧的对话历史压缩为摘要，释放上下文空间。

### 可用的 Compaction 模式

```json
// openclaw.json
{
  "agents": {
    "defaults": {
      "compaction": {
        "mode": "safeguard"
      }
    }
  }
}
```

| 模式 | 说明 |
|------|------|
| `safeguard` | 当上下文接近窗口上限时触发（唯一可用模式） |
| `off` | 关闭 compaction |

> **⚠️ OpenClaw 2026.2.9 已知问题：** `safeguard` 模式虽然是有效配置值，但在实际运行中 **从未成功触发 compaction**（所有 agent 的 `compactionCount` 均为 0）。`auto`、`rolling` 等值不被支持，设置后会导致 `Invalid config` 错误。

### Compaction 失效的影响

由于 safeguard compaction 不生效，所有持久会话的上下文会无限增长直到撑满窗口：

```
会话创建 → 正常使用 → 上下文逐渐增长 → 撑满 256K → 响应极慢/超时
                                                    ↑
                                            compaction 应在这里触发但没有
```

## 17.3 会话满载问题

### 症状

当会话上下文接近或达到模型窗口上限时，会出现以下问题：

| 症状 | 说明 |
|------|------|
| 响应变慢 | 正常 3-9 秒的请求变为 20-30 秒 |
| 极端超时 | 最慢可达 200+ 秒 |
| 飞书重试 | 飞书因超时自动重发请求，进一步加重负载 |
| 内容截断 | 模型可能因为上下文满载而省略部分回复 |

### 诊断方法

**方式一：通过 session-cleanup Skill**

```
session-cleanup status
session-cleanup overloaded
```

**方式二：手动查看 sessions.json**

```bash
python3 -c "
import json, os, time
agents_dir = os.path.expanduser('~/.openclaw/agents')
now_ms = time.time() * 1000
for agent_id in sorted(os.listdir(agents_dir)):
    sf = os.path.join(agents_dir, agent_id, 'sessions', 'sessions.json')
    if not os.path.exists(sf): continue
    with open(sf) as f: data = json.load(f)
    for key, meta in data.items():
        if not isinstance(meta, dict): continue
        total = meta.get('totalTokens', 0)
        ctx = meta.get('contextTokens', 0)
        pct = (total / ctx * 100) if ctx > 0 else 0
        if pct >= 80:
            ch = key.split(':')[2] if len(key.split(':')) >= 3 else '?'
            print(f'🔴 [{agent_id}] {ch}: {total:,}/{ctx:,} ({pct:.0f}%)')
"
```

### 实际案例

**案例：飞书群 broadband-ops 响应慢**

| 指标 | 异常值 |
|------|-------|
| totalTokens | 256,000（100%） |
| inputTokens | 414,584 |
| compactionCount | 0（compaction 从未触发） |
| 会话文件 | 631 KB / 302 轮对话 |
| 平均响应时间 | 24.4 秒 |
| 最大响应时间 | 201.9 秒 |

**重置后恢复：**

| 指标 | 重置后 |
|------|-------|
| totalTokens | 0（新会话） |
| 平均响应时间 | 7-9 秒 |
| 渠道绑定 | 不受影响，自动重建 |

## 17.4 session-cleanup Skill

### 概述

`session-cleanup` 是部署在 Agent Manager workspace 下的维护技能，提供两种会话维护能力：

| 维护类型 | 适用对象 | 触发条件 |
|----------|----------|----------|
| TTL 过期清理 | 临时会话（openai / cron） | 4 小时无活动 |
| 满载重置 | **所有会话**（含 telegram / feishu / main） | 上下文使用率 ≥80% |

### 命令列表

| 命令 | 说明 |
|------|------|
| `status` | 查看各 Agent 会话数量、磁盘占用和满载告警 |
| `dry-run` | 预览将要清理的过期临时会话（不删除） |
| `cleanup` | 执行清理过期临时会话（默认 TTL 4 小时） |
| `cleanup-ttl N` | 自定义 TTL 清理（N 为小时数） |
| `cleanup-agent X` | 只清理指定 Agent |
| `overloaded [N]` | 列出上下文 ≥N% 的满载会话（默认 80%） |
| `reset-overloaded [N]` | 重置满载会话（含受保护渠道会话） |
| `reset-overloaded-dry [N]` | 预览满载重置（不执行） |
| `auto` | 自动维护：清理过期 + 重置满载（crontab 调用） |
| `logs` | 查看自动维护历史日志 |

### 自动维护

已配置系统 crontab，每小时整点自动执行 `--auto` 模式：

```
0 * * * * /usr/bin/python3 /Users/<username>/.openclaw/workspace/agent-manager/skills/session-cleanup/scripts/session_cleanup.py --auto >> /Users/<username>/.openclaw/logs/session-cleanup.log 2>&1
```

> **注意：** crontab 中必须使用绝对路径，`~` 不会被展开。将 `<username>` 替换为实际用户名。

自动维护流程：

```
每小时整点
  ├── [1/2] 清理过期临时会话（TTL 4 小时）
  │     └── 仅清理 openai:* 和 cron:* 会话
  └── [2/2] 重置满载会话（阈值 80%）
        └── 清理所有上下文 ≥80% 的会话（含渠道持久会话）
```

### 满载重置阈值选择

默认阈值 80%（204K / 256K tokens），理由：

| 阈值 | 可用 tokens | 剩余缓冲 | 评估 |
|------|------------|----------|------|
| 70% | 179K | 77K（30%） | 偏激进，频繁重置影响体验 |
| **80%** | **204K** | **52K（20%）** | **均衡，推荐默认值** |
| 90% | 230K | 26K（10%） | 危险，一小时内可能冲到 100% |

### 安全措施

- 每次重置前自动备份（`.jsonl` 备份为 `.bak-before-reset`）
- 满载重置不影响渠道绑定，下次对话自动创建新会话
- Skill 包含完整安全规则（CONFIDENTIAL 标记 + SKILL.md 安全章节）

## 17.5 飞书渠道性能优化

### blockStreaming 设置

飞书渠道有两种消息推送模式：

| 设置 | 行为 | 适用场景 |
|------|------|---------|
| `blockStreaming: true` | 等待完整回复后一次性发送 | 回复较短时体验好 |
| `blockStreaming: false` | 流式逐步推送回复 | **推荐** — 用户可逐步看到回复 |

```json
// openclaw.json → channels.feishu
{
  "streaming": true,
  "blockStreaming": false
}
```

> **建议关闭 blockStreaming**：当 Agent 回复较长或 LLM 推理较慢时，`blockStreaming: true` 会让用户长时间看不到任何回复，体验很差。关闭后用户可以逐步看到回复内容。

### 飞书重试机制

飞书在约 19 秒内未收到响应时会自动重发请求。在 gateway 日志中表现为同一个请求出现两次 `dispatching to agent`，间隔约 19 秒。这是飞书平台行为，无法在 OpenClaw 侧控制。

当 Agent 响应慢于 19 秒时，飞书重试会导致重复处理，进一步增加负载。**保持会话健康（上下文不满载）是避免此问题的根本方法。**

## 17.6 响应慢排查流程

当用户反映某个渠道的 Agent 响应慢时，按以下步骤排查：

```
1. 检查会话上下文使用率
   └─ session-cleanup overloaded
   └─ 如果 ≥80%  → 重置会话（session-cleanup reset-overloaded）

2. 检查 blockStreaming 设置
   └─ 查看 openclaw.json → channels.<channel>.blockStreaming
   └─ 如果为 true → 改为 false，重启 gateway

3. 检查 gateway 日志
   └─ ~/.openclaw/logs/gateway.log — 请求延迟
   └─ ~/.openclaw/logs/gateway.err.log — 错误和异常

4. 检查模型提供商状态
   └─ 排除 LLM API 本身的延迟或限流问题
```

## 17.7 contextWindow 调优

### 原理

`contextWindow` 定义了 OpenClaw 向 LLM 发送的最大上下文长度。LLM 推理时间与输入 token 数近似成正比，减小 contextWindow 可直接缩短响应时间。

### 配置位置

contextWindow 在两个层级配置，优先级如下：

```
openclaw.json → models.providers.<provider>.models[].contextWindow   （全局）
agents/<id>/agent/models.json → providers.<provider>.models[].contextWindow （Agent 级）
```

> **⚠️ 避免两层同时配置。** `openclaw.json` 中 `models.mode: "merge"` 表示全局与 Agent 级合并，但 gateway 重启时可能用全局配置覆盖 Agent 级文件。**建议只在全局 `openclaw.json` 中统一配置，删除 Agent 级的 `models.json`**，避免冲突。

### 推荐值

| contextWindow | 可存对话 | 80% 自动重置点 | 响应速度 | 适合场景 |
|---------------|---------|---------------|---------|---------|
| 256K | 数百轮 | 204K tokens | 较慢 | 长期深度分析、复杂多轮推理 |
| **128K** | **百余轮** | **102K tokens** | **较快** | **日常运维群聊（当前部署值）** |
| 64K | 数十轮 | 51K tokens | 最快 | 指令型运维交互（Skill 少、对话短时可用） |

### 当前部署

```json
// openclaw.json → models.providers.volcengine
{
  "id": "ark-code-latest",
  "contextWindow": 128000,   // 从 256K 调至 128K
  "maxTokens": 8192           // 不变
}
```

影响的 Agent：broadband-ops、ops（均使用 volcengine/ark-code-latest）。

> **注意：** `maxTokens` 限制单次回复的最大 token 数，对响应速度影响不大（大部分回复远不到 8192 tokens），建议保持不变。
>
> **为什么选 128K 而非 64K？** 当前 broadband-ops 有 7 个 Skill，系统提示（AGENTS.md + TOOLS.md 等）约占 7K-12K tokens，64K 窗口在多 Skill 连续执行的复杂会话中偏紧张，128K 提供了充足的余量。

## 17.8 模型选择建议

为 Agent 选择模型时，根据工作负载特征匹配：

| 工作负载特征 | 推荐模型方向 | 不推荐 |
|-------------|------------|--------|
| 多工具调度、工作流编排 | Agentic 优化模型（如 Doubao-Seed-Code） | 前端/设计优化模型 |
| 需要快速响应 | non-thinking 默认模型 | thinking 模型（延迟高） |
| 运维/后端操作 | 代码/推理模型 | 前端优化模型（如 Kimi-K2.5） |
| 简单日常问答 | 轻量级通用模型（如 Deepseek v3.2） | 重量级推理模型 |

> **关键原则：** 运维类 Agent（如 broadband-ops）优先选择 Agentic 编程优化模型，它们在任务调度、逻辑协同、工具调用方面表现更好。thinking 模式虽然推理更强，但会显著增加延迟，不适合需要快速响应的运维场景。

## 17.9 并发数调优

### 配置位置

```json
// openclaw.json → agents.defaults
{
  "maxConcurrent": 4,
  "subagents": {
    "maxConcurrent": 8
  }
}
```

| 参数 | 说明 |
|------|------|
| `maxConcurrent` | 最大同时活跃的会话数 |
| `subagents.maxConcurrent` | 最大同时活跃的子 agent 数 |

### 推荐值

并发数应匹配实际 Agent 规模，过高的并发会导致本地资源过度消耗、API 限流：

```
maxConcurrent ≈ Agent 数量 × 1.5（向上取整）
subagents.maxConcurrent ≈ maxConcurrent × 2
```

| Agent 数量 | maxConcurrent | subagents |
|------------|---------------|-----------|
| 1-3 | 4 | 8 |
| 4-6 | 8 | 16 |
| 7-10 | 12 | 24 |

> 超过并发限制的请求会排队等待，对日常使用无感知。只在极端并发场景（如 cron 任务集中触发）才可能出现排队。

### 当前部署

3 个 Agent（ops、agent-manager、broadband-ops）→ `maxConcurrent: 4`，`subagents: 8`。

## 17.10 模型参数限制

### temperature — 不支持在 openclaw.json 中配置

尝试在 `models.providers.*.models[]` 中添加 `"temperature"` 参数，OpenClaw 启动时会报错：

```
Unrecognized key: "temperature"
```

`temperature` 不在 OpenClaw 的模型配置 schema 中。如需调整模型生成的随机性，可能的替代方案：

- 在模型提供商侧设置默认 temperature
- 等待 OpenClaw 未来版本支持此参数
- 通过 Agent 的 system prompt（AGENTS.md / SOUL.md）引导模型行为的确定性

### contextWindow 与 Skill 数量的关系

Skill 数量增加会从两个层面影响上下文消耗：

**固定开销（每次对话都加载）：**

TOOLS.md 包含所有 Skill 的使用说明，Skill 越多 TOOLS.md 越大。

| 系统文件 | 作用 | 典型大小 |
|----------|------|----------|
| AGENTS.md | Agent 宪法、安全规则 | 3K-5K tokens |
| TOOLS.md | 所有 Skill 的说明 | 3K-5K tokens（7 个 Skill 时） |
| 其他系统文件 | SOUL/MEMORY/IDENTITY 等 | 1K-2K tokens |
| **合计** | | **7K-12K tokens** |

**按需开销（执行 Skill 时加载）：**

| 环节 | 估算 tokens |
|------|------------|
| 读取 SKILL.md | 500-4K（取决于 Skill 复杂度） |
| 脚本执行输出 | 500-5K（取决于查询结果大小） |
| 交互对话 | 500-1K |

**优化方向：** 当 Skill 数量大幅增加时，优先精简 TOOLS.md 中的描述（减少固定开销），而非盲目加大 contextWindow。
