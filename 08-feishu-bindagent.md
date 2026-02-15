# 8. 为飞书绑定 Agent

## 8.1 绑定方式

飞书支持两种绑定方式：

### 方式一：通过 bindings 绑定（推荐）

在 `openclaw.json` 的 `bindings` 数组中添加：

```jsonc
"bindings": [
  {
    "agentId": "broadband-ops",
    "match": {
      "channel": "feishu",
      "peer": {
        "kind": "group",
        "id": "<飞书群组 open_id>"     // 如 "oc_xxxxxxxxxxxx..."
      }
    }
  }
]
```

### 方式二：通过 groups 配置绑定

在 `channels.feishu.groups` 中直接指定：

```jsonc
"channels": {
  "feishu": {
    "groups": {
      "<飞书群组 open_id>": {
        "enabled": true,
        "requireMention": true,
        "agent": "broadband-ops"      // 直接在群组配置中指定 Agent
      }
    }
  }
}
```

### 两种方式的区别

| 特性 | bindings 方式 | groups 方式 |
|------|:---:|:---:|
| 统一管理所有渠道绑定 | ✅ | ❌ |
| 精细控制群组行为 | ❌ | ✅ |
| 可设置 requireMention | ❌ | ✅ |
| 支持默认路由 | ✅ | ❌ |

**建议：** 两种方式可以同时使用。`groups` 中设置群组行为（如 `requireMention`），`bindings` 中设置路由规则。

## 8.2 设置飞书默认路由

通过 `channels.feishu.agent` 设置未匹配群组的默认 Agent：

```jsonc
"channels": {
  "feishu": {
    "agent": "broadband-ops"    // 飞书默认 Agent
  }
}
```

也可以通过 bindings 设置兜底：

```jsonc
{
  "agentId": "agent-manager",
  "match": {
    "channel": "feishu"           // 无 peer → 匹配所有未绑定的飞书消息
  }
}
```

## 8.3 完整飞书绑定示例

```jsonc
// channels.feishu 配置
"feishu": {
  "enabled": true,
  "domain": "feishu",
  "dmPolicy": "allowlist",                       // 仅白名单用户可私聊
  "allowFrom": ["ou_xxxxxxxxxxxx"],              // 管理员的飞书 open_id
  "groupPolicy": "allowlist",                    // 仅允许指定群组
  "groupAllowFrom": ["oc_xxxxxxxxxxxx"],         // 允许的群组 ID 列表
  "streaming": true,
  "blockStreaming": true,
  "accounts": {
    "main": {
      "appId": "<App ID>",
      "appSecret": "<App Secret>",
      "botName": "飞书AI助手"
    }
  },
  "groups": {
    "oc_xxxxxxxxxxxx": {                         // 群组精细配置
      "enabled": true,
      "requireMention": true,
      "agent": "broadband-ops"
    }
  },
  "agent": "broadband-ops"                      // 默认 Agent
}

// bindings 配置
"bindings": [
  {
    "agentId": "broadband-ops",
    "match": {
      "channel": "feishu",
      "peer": { "kind": "group", "id": "oc_xxxxxxxxxxxx" }
    }
  },
  {
    "agentId": "agent-manager",
    "match": {
      "channel": "feishu"                        // 兜底
    }
  }
]
```

> **注意**：`groupAllowFrom` 中的群组 ID 和 `groups` 中的群组 ID 都是 `oc_xxx` 格式。两者需保持一致 — `groupAllowFrom` 控制准入，`groups` 控制行为。详见 [第 7 章 groupAllowFrom 说明](./07-feishu.md)。

## 8.4 Telegram + 飞书混合绑定

一个 Agent 可以同时绑定到 Telegram 和飞书群组：

```jsonc
"bindings": [
  // BroadbandOpsBot 同时绑定 Telegram 和飞书
  {
    "agentId": "broadband-ops",
    "match": {
      "channel": "telegram",
      "peer": { "kind": "group", "id": "<Telegram群组ID>" }
    }
  },
  {
    "agentId": "broadband-ops",
    "match": {
      "channel": "feishu",
      "peer": { "kind": "group", "id": "<飞书群组open_id>" }
    }
  },
  // 各渠道兜底
  {
    "agentId": "agent-manager",
    "match": { "channel": "telegram" }
  },
  {
    "agentId": "agent-manager",
    "match": { "channel": "feishu" }
  }
]
```

## 8.5 ACK 反馈配置

控制 Bot 收到消息后的确认行为：

```jsonc
"messages": {
  "ackReactionScope": "group-mentions"   // 仅在群组被 @时 发送确认反应
}
```

| 值 | 行为 |
|----|------|
| `"all"` | 所有消息都发送确认 |
| `"group-mentions"` | 仅群组 @消息发送确认 |
| `"none"` | 不发送确认 |

## 8.6 群内多人并发操作的确认安全

群聊中多人同时操作不同 Skill 时，确认消息可能被 LLM 误关联到其他用户的操作，导致误执行。这是群聊场景下的重要安全问题。

**核心风险**：群内所有用户共享同一个会话上下文，而确认机制是纯提示词约束，LLM 无法验证"回复确认的人"是否就是"发起操作的人"。

**已实施的解决方案**：所有 Skill 的确认已改为 **特定确认短语机制**（如"确认删除 SN 123"、"确认回单 100235"），拒绝通用的"确认"回复。

> 📖 **完整的问题分析、解决方案和操作规范详见 [第 19 章：群聊多人并发操作的确认安全](./19-group-confirmation-safety.md)**
