# 8. 为飞书绑定 Agent

## 8.1 绑定方式

当前配置中，Agent 路由统一走 `bindings`；`groups` 用于群组行为控制（如 `requireMention`）。

### 方式一：通过 bindings 绑定（推荐）

在 `openclaw.json` 的 `bindings` 数组中添加：

```jsonc
"bindings": [
  {
    "agentId": "broadband-ops",
    "match": {
      "channel": "feishu",
      "accountId": "main",                 // 与 accounts.main 对应；和 peer 同级
      "peer": {
        "kind": "group",
        "id": "<飞书群组 open_id>"     // 如 "oc_xxxxxxxxxxxx..."
      }
    }
  }
]
```

### 方式二：通过 groups 配置群组行为（非路由）

在 `channels.feishu.groups` 中设置群行为：

```jsonc
"channels": {
  "feishu": {
    "groups": {
      "<飞书群组 open_id>": {
        "enabled": true,
        "requireMention": true
      }
    }
  }
}
```

### 两种方式的区别

| 特性 | bindings 方式 | groups 方式 |
|------|:---:|:---:|
| Agent 路由匹配 | ✅ | ❌ |
| `accountId` 匹配 | ✅ | ❌ |
| 精细控制群组行为 | ❌ | ✅ |
| 可设置 `requireMention` | ❌ | ✅ |

**建议：** `groups` 用于行为控制，`bindings` 用于路由控制。多账号场景请在 `bindings.match.accountId` 显式配置账号名。

## 8.2 设置飞书默认路由

如果你使用多账号（如 `accounts.main`），通过 `bindings` 做飞书兜底时也要写 `match.accountId`，否则只会匹配 `default` 账号。

当前配置更推荐通过 `bindings` 设置飞书兜底：

```jsonc
{
  "agentId": "agent-manager",
  "match": {
    "channel": "feishu",
    "accountId": "main"           // 多账号时需要显式指定
  }
}
```

## 8.3 完整飞书绑定示例

```jsonc
// channels.feishu 配置
"feishu": {
  "enabled": true,
  "domain": "feishu",
  "dmPolicy": "disabled",                        // 当前配置：关闭私聊
  "allowFrom": [],
  "groupPolicy": "allowlist",                    // 仅允许指定群组
  "groupAllowFrom": [
    "oc_xxxxxxxxxxxx",
    "oc_yyyyyyyyyyyy"
  ],
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
      "requireMention": true
    },
    "oc_yyyyyyyyyyyy": {
      "enabled": true,
      "requireMention": true
    }
  }
}

// bindings 配置
"bindings": [
  {
    "agentId": "broadband-ops",
    "match": {
      "channel": "feishu",
      "accountId": "main",
      "peer": { "kind": "group", "id": "oc_xxxxxxxxxxxx" }
    }
  },
  {
    "agentId": "agent-manager",
    "match": {
      "channel": "feishu",
      "accountId": "main",
      "peer": { "kind": "group", "id": "oc_yyyyyyyyyyyy" }
    }
  }
]
```

> **注意**：`groupAllowFrom` 中的群组 ID 和 `groups` 中的群组 ID 都是 `oc_xxx` 格式。两者需保持一致 — `groupAllowFrom` 控制准入，`groups` 控制行为。详见 [第 7 章 groupAllowFrom 说明](./07-feishu.md)。

### accountId 路由命中（常见坑）

### 根因说明

- 当飞书使用 `channels.feishu.accounts.main` 这类多账号配置时，绑定若未写 `match.accountId: "main"`，会因账号不匹配而 miss，最终回落到默认 Agent。
- 路由规则是统一的：`bindings.match.accountId` 省略时，仅匹配 `default` 账号。
- `accountId` 必须写在 `match` 下，并且与 `peer` 同级；不能写在 `match.peer` 内。

### 为什么 Telegram 常见场景“看起来不用写 accountId”

- 若 Telegram 使用的是顶层单账号配置（未使用 `accounts`），当前账号即 `default`。
- 此时省略 `match.accountId` 仍可匹配，所以看起来“无需填写”。

### 三种账号配置与绑定写法

1. 顶层单账号（不写 `accounts`）  
   - 账号名为 `default`；`bindings` 可省略 `match.accountId`。
2. 多账号（`accounts.main` / `accounts.xxx`）  
   - `bindings` 应写对应 `match.accountId`（如 `"main"` / `"xxx"`）。
3. 顶层 + 账号覆盖（merge）  
   - 绑定仍按最终生效账号匹配，建议显式写 `match.accountId` 以避免歧义。

### 可直接使用的飞书绑定示例

```jsonc
"bindings": [
  {
    "agentId": "broadband-ops",
    "match": {
      "channel": "feishu",
      "accountId": "main",
      "peer": { "kind": "group", "id": "oc_xxxxxxxxxxxx" }
    }
  }
]
```

> **安全提醒：** 若配置文件中出现了真实 `appSecret` / token，请尽快在平台侧轮换并改用安全存储方式（如环境变量或密钥管理）。

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
      "accountId": "main",   // 多账号时显式指定
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
    "match": { "channel": "feishu", "accountId": "main" }
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
