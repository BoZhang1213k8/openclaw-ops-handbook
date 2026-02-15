# 6. 创建 Telegram 群组并绑定 Agent

## 6.1 创建群组并添加 Bot

1. 在 Telegram 中创建新群组
2. 将你的 Bot 添加为群组成员
3. **获取群组 ID**：
   - 在群组中发送一条消息
   - 查看网关日志获取群组 ID：
     ```bash
     tail -f ~/.openclaw/logs/gateway.log | grep "group"
     ```
   - 群组 ID 通常以 `-` 开头（如 `-512345xxxx`）

## 6.2 添加群组白名单

在 `openclaw.json` 的 `channels.telegram.groups` 中添加：

```jsonc
"channels": {
  "telegram": {
    "groups": {
      "<群组ID>": {
        "requireMention": false    // false = 不需要 @Bot 即可响应
      }
    }
  }
}
```

**`requireMention` 说明：**

| 值 | 行为 |
|----|------|
| `false` | Bot 响应群组中的所有消息 |
| `true` | 只在被 @提及 时才响应 |

**建议：**
- 专属群组（一个 Agent 一个群）设为 `false`
- 多人活跃群组设为 `true`，避免干扰

## 6.3 绑定群组到 Agent

在 `openclaw.json` 的 `bindings` 数组中添加绑定规则：

```jsonc
"bindings": [
  {
    "agentId": "ops",
    "match": {
      "channel": "telegram",
      "peer": {
        "kind": "group",
        "id": "<群组ID>"
      }
    }
  }
]
```

**绑定规则说明：**

| 字段 | 说明 |
|------|------|
| `agentId` | 要绑定的 Agent ID |
| `match.channel` | 渠道类型（`telegram` / `feishu`） |
| `match.peer.kind` | 对话类型（`group` = 群组） |
| `match.peer.id` | 群组 ID |

## 6.4 设置默认路由（兜底）

可以设置一个默认的 Telegram 路由，处理未匹配任何群组的消息：

```jsonc
{
  "agentId": "agent-manager",
  "match": {
    "channel": "telegram"          // 没有 peer → 匹配所有未绑定的 Telegram 消息
  }
}
```

> **注意：** 默认路由放在 `bindings` 数组的**最后**，因为匹配是按顺序进行的。

## 6.5 完整绑定示例

```jsonc
"bindings": [
  // 精确匹配：OpsBot 群组
  {
    "agentId": "ops",
    "match": {
      "channel": "telegram",
      "peer": { "kind": "group", "id": "<ops群组ID>" }
    }
  },
  // 精确匹配：家宽运维群组
  {
    "agentId": "broadband-ops",
    "match": {
      "channel": "telegram",
      "peer": { "kind": "group", "id": "<broadband-ops群组ID>" }
    }
  },
  // 默认路由：Agent Manager 处理其余消息
  {
    "agentId": "agent-manager",
    "match": {
      "channel": "telegram"
    }
  }
]
```

## 6.6 匹配优先级

绑定按数组顺序匹配，第一个匹配的规则生效：

```
消息进入
  ↓
遍历 bindings 数组
  ↓
[1] channel=telegram, peer.id=-xxx1 → 匹配 OpsBot       → 路由到 OpsBot
[2] channel=telegram, peer.id=-xxx2 → 匹配 BroadbandOps → 路由到 BroadbandOps
[3] channel=telegram (无 peer)      → 兜底匹配           → 路由到 AgentManager
```

## 6.7 重启生效

```bash
openclaw gateway restart
```

然后在对应群组中发送消息测试，确认消息被正确路由到了目标 Agent。
