# 5. 接入 Telegram 机器人

## 5.1 创建 Telegram Bot

1. 在 Telegram 中搜索 `@BotFather`
2. 发送 `/newbot`
3. 按提示设置 Bot 名称和用户名
4. 获取 Bot Token（格式如 `1234567890:AAxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`）

## 5.2 配置 Bot Token

在 `openclaw.json` 的 `channels.telegram` 中配置：

```jsonc
"channels": {
  "telegram": {
    "enabled": true,
    "botToken": "<你的 Bot Token>",
    "dmPolicy": "disabled",         // 私聊策略: disabled（关闭私聊）
    "groupPolicy": "allowlist",     // 群组策略: allowlist（白名单）
    "streamMode": "partial",        // 流式输出模式
    "groups": {
      // 稍后添加群组白名单
    }
  }
}
```

**策略说明：**

| 配置 | 可选值 | 说明 |
|------|--------|------|
| `dmPolicy` | `pairing` / `allowlist` / `open` / `disabled` | 私聊访问控制策略 |
| `groupPolicy` | `allowlist` / `open` | `allowlist` = 只允许白名单群组 |
| `streamMode` | `partial` / `full` / `off` | 控制流式输出行为 |

**dmPolicy 详解：**

| 值 | 行为 | 适用场景 |
|------|------|----------|
| `"pairing"` | 陌生人私聊需配对码审批 | 默认值，需要私聊但要控制的场景 |
| `"allowlist"` | 只有 `allowFrom` 列表中的用户能私聊 | 管理员专用私聊 |
| `"open"` | 任何人都能私聊 | 不推荐 |
| `"disabled"` | 完全关闭私聊 | **推荐用于运维场景** — 只需群聊交互 |

> **安全建议：** Telegram bot 是公开的，任何知道 bot 用户名的人都能尝试发起私聊。对于运维类 Agent，建议将 `dmPolicy` 设为 `"disabled"`，仅通过白名单群组交互，避免未授权访问。

## 5.3 启用 Telegram 插件

在 `openclaw.json` 的 `plugins` 中启用：

```jsonc
"plugins": {
  "entries": {
    "telegram": {
      "enabled": true
    }
  }
}
```

## 5.4 配置 Telegram 用户白名单（可选）

如需限制谁可以与 Bot 交互，编辑 `~/.openclaw/credentials/telegram-allowFrom.json`：

```jsonc
{
  "userIds": [
    123456789    // 你的 Telegram User ID
  ]
}
```

## 5.5 代理配置（中国大陆需要）

如在中国大陆使用，需要配置代理：

```jsonc
"env": {
  "HTTPS_PROXY": "http://127.0.0.1:7890",
  "HTTP_PROXY": "http://127.0.0.1:7890",
  "ALL_PROXY": "socks5://127.0.0.1:7891",
  "NO_PROXY": "api.moonshot.cn,volces.com"
}
```

> **注意：** `NO_PROXY` 中应包含国内模型 API 的域名，避免国内 API 走代理。

## 5.6 重启网关

配置完成后重启网关使配置生效：

```bash
openclaw gateway restart
```

## 5.7 验证连接

1. 在 Telegram 中找到你的 Bot
2. 发送 `/start`
3. 如果 `dmPolicy` 为 `pairing`，需要先完成设备配对
4. 查看网关日志确认连接：

```bash
tail -f ~/.openclaw/logs/gateway.log
```
