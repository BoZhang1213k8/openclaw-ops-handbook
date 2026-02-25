# 13. 定时任务（Cron）配置

## 本章定位（把人工操作变成自动运维）

完成前面章节后，你已经具备 Agent、渠道、Skill 和权限策略。  
本章把这些能力编排为可定时执行的任务，适用于巡检、报表、通知等周期性工作。

建议顺序：先通过 Agent Manager 创建一条任务，再回看 `jobs.json` 字段细节，最后补权限校验。

## 13.1 概述

OpenClaw 支持 Cron 定时任务，可以让 Agent 按计划自动执行任务并将结果推送到指定渠道。

配置文件：`~/.openclaw/cron/jobs.json`

## 13.2 配置结构

```jsonc
{
  "version": 1,
  "jobs": [
    {
      "id": "<UUID>",                    // 任务唯一 ID
      "agentId": "ops",                  // 执行 Agent
      "name": "主机巡检日报",             // 任务名称
      "enabled": true,                   // 是否启用
      "schedule": {
        "kind": "cron",
        "expr": "0 9 * * *",            // Cron 表达式
        "tz": "Asia/Shanghai"            // 时区
      },
      "sessionTarget": "isolated",       // 会话模式
      "wakeMode": "now",                 // 唤醒模式
      "payload": {
        "kind": "agentTurn",
        "message": "执行主机巡检并推送结果",  // 发送给 Agent 的指令
        "timeoutSeconds": 120            // 超时时间
      },
      "delivery": {
        "mode": "announce",
        "to": "<渠道群组ID>",            // 推送目标
        "channel": "telegram"            // 推送渠道
      }
    }
  ]
}
```

## 13.3 字段说明

### schedule（调度配置）

| 字段 | 说明 | 示例 |
|------|------|------|
| `kind` | 调度类型 | `"cron"` |
| `expr` | Cron 表达式 | `"0 9 * * *"` = 每天 9:00 |
| `tz` | 时区 | `"Asia/Shanghai"` |

**常用 Cron 表达式：**

| 表达式 | 含义 |
|--------|------|
| `0 9 * * *` | 每天 09:00 |
| `0 9 * * 1-5` | 工作日 09:00 |
| `0 */2 * * *` | 每 2 小时 |
| `*/30 * * * *` | 每 30 分钟 |
| `0 9,18 * * *` | 每天 09:00 和 18:00 |

### payload（任务内容）

| 字段 | 说明 |
|------|------|
| `kind` | 固定为 `"agentTurn"` |
| `message` | 发送给 Agent 的消息（Agent 会据此执行操作） |
| `timeoutSeconds` | 任务超时时间（秒） |

### delivery（推送配置）

| 字段 | 说明 |
|------|------|
| `mode` | `"announce"` = 推送结果 |
| `to` | 目标群组 ID |
| `channel` | 推送渠道（`"telegram"` / `"feishu"`） |

### sessionTarget（会话模式）

| 值 | 说明 |
|----|------|
| `"isolated"` | 每次执行创建独立会话（推荐） |
| `"current"` | 使用当前活跃会话 |

## 13.4 创建定时任务

### 通过 Agent Manager 创建

在 Agent Manager 的 Telegram/飞书群组中：

```
设置一个定时任务：每天早上 9 点，让 OpsBot 执行主机巡检，结果推送到 OpsBot 的 Telegram 群组
```

Agent Manager 会使用 cron 工具创建任务。

### 手动编辑 jobs.json

也可以直接编辑 `~/.openclaw/cron/jobs.json`。

> **注意：** `id` 字段需要是 UUID 格式，可以用 `uuidgen` 命令生成。

## 13.5 任务状态

每个任务有 `state` 字段跟踪执行状态：

```jsonc
"state": {
  "nextRunAtMs": 1739336400000,     // 下次执行时间
  "lastRunAtMs": 1739250000000,     // 上次执行时间
  "lastStatus": "ok",               // 上次状态
  "lastDurationMs": 45000,          // 上次耗时
  "consecutiveErrors": 0            // 连续错误次数
}
```

## 13.6 权限配置

确保 `policy.yaml` 中为需要管理 cron 的 Agent 开放权限：

```yaml
tools:
  cron:
    deny: ["*"]
    allow: ["agent-manager", "ops"]
```

## 13.7 示例：每日巡检任务

```jsonc
{
  "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "agentId": "ops",
  "name": "主机巡检日报",
  "enabled": true,
  "schedule": {
    "kind": "cron",
    "expr": "0 9 * * *",
    "tz": "Asia/Shanghai"
  },
  "sessionTarget": "isolated",
  "wakeMode": "now",
  "payload": {
    "kind": "agentTurn",
    "message": "执行主机巡检 host-monitor 技能，使用 send-dingtalk 命令推送结果到钉钉群",
    "timeoutSeconds": 120
  },
  "delivery": {
    "mode": "announce",
    "to": "<Telegram群组ID>",
    "channel": "telegram"
  }
}
```

## 下一步

继续 [第 17 章：会话管理与性能优化](./17-session-management.md)，确保定时任务与多渠道长期运行时仍保持稳定响应。
