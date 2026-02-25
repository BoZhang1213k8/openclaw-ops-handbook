# 7. 接入飞书

## 7.1 创建飞书应用

1. 登录 [飞书开放平台](https://open.feishu.cn/)
2. 创建企业自建应用
3. 获取 `App ID` 和 `App Secret`
4. 在应用中启用「机器人」能力
5. **先发布一个版本**（创建版本 → 申请发布 → 审核通过）

> **重要：** 第 5 步必须在配置事件订阅之前完成。飞书要求应用至少发布过一个版本后，才能建立长连接。如果跳过此步骤直接配置事件订阅，会收到 **"应用未建立长连接"** 的错误提示。

### 所需权限

在飞书应用的权限管理中申请以下权限（发布前配置好）：

| 权限 | 说明 |
|------|------|
| `im:message` | 获取与发送消息 |
| `im:message.group_at_msg` | 接收群组 @消息 |
| `im:chat` | 获取群组信息 |

## 7.2 配置飞书渠道

在 `openclaw.json` 的 `channels.feishu` 中配置：

```jsonc
"channels": {
  "feishu": {
    "enabled": true,
    "domain": "feishu",                          // "feishu" 或 "lark"（国际版用 lark）
    "dmPolicy": "disabled",                      // 当前配置: 关闭私聊
    "allowFrom": [],                             // dmPolicy=disabled 时通常为空
    "groupPolicy": "allowlist",                  // 群组策略: allowlist = 仅允许指定群组
    "groupAllowFrom": ["oc_xxxxxxxxxxxx", "oc_yyyyyyyyyyyy"], // 允许的群组 ID 列表
    "streaming": true,                           // 启用流式输出
    "blockStreaming": true,                      // 阻塞式流式输出
    "accounts": {
      "main": {
        "appId": "<你的 App ID>",
        "appSecret": "<你的 App Secret>",
        "botName": "飞书AI助手"                   // Bot 显示名称
      }
    },
    "groups": {
      "oc_xxxxxxxxxxxx": {                       // 群组精细配置
        "enabled": true,
        "requireMention": true                   // 需要 @Bot 才响应
      },
      "oc_yyyyyyyyyyyy": {
        "enabled": true,
        "requireMention": true
      }
    }
  }
}
```

### dmPolicy 详解

| 值 | 行为 | 适用场景 |
|------|------|----------|
| `"pairing"` | 陌生人私聊需配对码审批 | 默认值，需要私聊但要控制的场景 |
| `"allowlist"` | 只有 `allowFrom` 列表中的用户能私聊 | **推荐** — 管理员专用私聊 |
| `"open"` | 企业内任何人都能私聊 | 不推荐 |
| `"disabled"` | 完全关闭私聊 | 只需群聊的场景 |

> **安全建议：** 飞书 bot 虽然限定在企业内部，但组织内任何员工都能搜到 bot 并发起私聊。建议将 `dmPolicy` 设为 `"allowlist"`，仅允许管理员私聊 bot。

### allowFrom 配置说明

`allowFrom` 是飞书用户 `open_id` 的数组（格式为 `ou_xxx`），用于配合 `dmPolicy: "allowlist"` 使用。

**获取飞书 open_id 的方式：**

1. 飞书管理后台 → 成员管理 → 查看用户详情
2. 查看 OpenClaw 网关日志（日志中会记录消息发送者的 `open_id`）：

```bash
grep "received message from" ~/.openclaw/logs/gateway.log | grep feishu
```

3. 查看 OpenClaw 会话记录中的用户标识

### groupPolicy 说明

| 值 | 行为 | 说明 |
|------|------|------|
| `"open"` | 允许所有群组的所有人 | 默认值，机器人被拉入任何群都会响应 |
| `"allowlist"` | 仅允许 `groupAllowFrom` 中指定的群组 | **推荐** — 防止机器人被拉入未授权的群 |
| `"disabled"` | 完全禁用群组消息 | 只需私聊的场景 |

> **安全建议：** 飞书企业内的任何成员都可以将 Bot 拉入自己的群聊。如果 `groupPolicy` 为 `"open"`，Bot 会在所有群中响应，可能导致运维能力被非授权人员使用。**建议设为 `"allowlist"` 并通过 `groupAllowFrom` 限制为指定群组。**

### groupAllowFrom 配置说明

`groupAllowFrom` 是飞书**群组 ID** 的数组（格式为 `oc_xxx`），用于配合 `groupPolicy: "allowlist"` 使用。

> **关键：** `groupAllowFrom` 中放的是**群组 ID**（`oc_xxx`），不是用户 ID（`ou_xxx`）。这与 `allowFrom`（用于 `dmPolicy`，存放用户 ID）不同。

**配置示例 — 仅允许一个群组：**

```jsonc
"channels": {
  "feishu": {
    "groupPolicy": "allowlist",
    "groupAllowFrom": ["oc_xxxxxxxxxxxx"],  // 允许的群组 ID 列表
    "groups": {
      "oc_xxxxxxxxxxxx": {
        "enabled": true,
        "requireMention": true,
        "agent": "broadband-ops"
      }
    }
  }
}
```

**群内用户级别的进一步限制（可选）：**

如果还需要限制群内谁可以触发机器人，可在 `groups.<chat_id>.allowFrom` 中配置**用户 ID**：

```jsonc
"groups": {
  "oc_xxxxxxxxxxxx": {
    "enabled": true,
    "requireMention": true,
    "agent": "broadband-ops",
    "allowFrom": ["ou_aaa", "ou_bbb"]  // 仅这些用户可在此群中触发机器人
  }
}
```

### 访问控制两层模型

```
第 1 层：groupPolicy + groupAllowFrom
  → 控制哪些「群组」可以使用机器人（群组 ID 级别）
  → groupAllowFrom 中填写 oc_xxx（群组 ID）

第 2 层：groups.<chat_id>.allowFrom（可选）
  → 控制群内哪些「用户」可以触发机器人（用户 ID 级别）
  → allowFrom 中填写 ou_xxx（用户 ID）
  → 不配置则群内所有人都可触发
```

### 获取飞书群组 ID

群组 ID 格式为 `oc_xxx`，获取方式：

1. **推荐**：启动网关后在群组中 @机器人发消息，查看日志：
   ```bash
   grep "received message from" ~/.openclaw/logs/gateway.log | grep feishu
   ```
   日志中会显示 `in oc_xxx (group)` 形式的群组 ID。

2. 使用飞书开放平台 API 调试工具获取机器人所在群组列表。

**飞书与 Telegram 配置差异：**

| 配置 | Telegram | 飞书 |
|------|----------|------|
| 认证方式 | Bot Token | App ID + App Secret |
| 群组白名单字段 | `groups` 中列出群 ID 即可 | `groupAllowFrom`（群组 ID 数组） |
| 群内用户限制 | 不支持 | `groups.<id>.allowFrom`（可选） |
| 流式输出 | `streamMode` | `streaming` + `blockStreaming` |
| 默认 Agent | 通过 bindings 设置 | `agent` 字段 |
| 安全边界 | 无（公网可访问） | 企业组织内 |
| dmPolicy 推荐 | `"disabled"` | `"allowlist"` |
| groupPolicy 推荐 | `"allowlist"` | `"allowlist"` |

## 7.3 启用飞书插件

```jsonc
"plugins": {
  "entries": {
    "feishu": {
      "enabled": true
    }
  }
}
```

## 7.4 配置事件订阅（长连接模式）

OpenClaw 使用飞书的**长连接（WebSocket）** 方式接收事件，无需配置 Webhook 回调地址，也无需服务器对外暴露端口。

### 长连接 vs Webhook

| 特性 | 长连接（OpenClaw 使用） | Webhook |
|------|:---:|:---:|
| 需要公网 IP / 域名 | ❌ 不需要 | ✅ 需要 |
| 需要反向代理 | ❌ 不需要 | ✅ 需要 |
| 适合内网部署 | ✅ 完美适配 | ❌ 需额外配置 |
| 连接方式 | 应用主动连接飞书服务器 | 飞书服务器回调应用 |

### 配置步骤

> **操作顺序很重要：** 必须先完成本地 OpenClaw 配置并启动网关、建立长连接后，再到飞书开放平台配置事件订阅。否则飞书会提示 **"应用未建立长连接"**。

**正确的流程：**

```
1. 飞书开放平台：创建应用、启用机器人、配置权限、发布版本
   ↓
2. 本地 OpenClaw：配置 channels.feishu（App ID、App Secret）→ 7.2
   ↓
3. 本地 OpenClaw：启用飞书插件 → 7.3、启动/重启网关
   ↓
4. 确认网关日志中显示飞书长连接已建立
   ↓
5. 飞书开放平台：事件订阅 → 选择「长连接」方式 → 添加事件
```

### 需要订阅的事件

在飞书开放平台的「事件订阅」页面，接收方式选择 **「使用长连接接收事件」**，然后添加以下事件：

| 事件 | 说明 |
|------|------|
| `im.message.receive_v1` | 接收消息 |

### 验证长连接

启动网关后，查看日志确认连接状态：

```bash
tail -f ~/.openclaw/logs/gateway.log | grep -i feishu
```

看到类似 `feishu connected` 或 `websocket established` 的日志即表示长连接建立成功。

## 7.5 飞书群组配置（精细控制）

如需对特定群组做精细控制：

```jsonc
"channels": {
  "feishu": {
    "groups": {
      "<群组 open_id>": {
        "enabled": true,
        "requireMention": true     // 需要 @Bot 才响应
      }
    }
  }
}
```

> **路由命中提示：** `groups` 只控制群行为（如 `requireMention`），不负责 Agent 路由。若使用多账号（如 `channels.feishu.accounts.main`），应在 `bindings[].match.accountId`（与 `match.peer` 同级）里写 `"main"`，否则绑定可能 miss 并回落到默认 Agent。详见 [第 8 章绑定示例](./08-feishu-bindagent.md)。

**获取飞书群组 open_id：**
- 查看网关日志
- 或使用飞书开放平台 API

## 7.6 重启并验证

```bash
openclaw gateway restart
```

1. 将飞书 Bot 添加到群组
2. 在群组中 @Bot 发送消息
3. 检查网关日志确认连接正常
