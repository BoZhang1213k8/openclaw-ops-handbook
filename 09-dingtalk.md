# 9. 接入钉钉自定义机器人

## 本章定位（通知通道，不是对话通道）

本章解决的是“Skill 执行结果如何主动推送到钉钉群”。  
与 Telegram/飞书不同，钉钉自定义机器人不参与 Agent 对话路由，而是作为 Skill 的输出目标。

建议顺序：创建钉钉机器人 → 在 Skill `.env` 配置 webhook/secret → 在 `skill.json` 增加 `send-*` 命令 → 脚本实现推送。

## 9.1 与 Telegram/飞书的区别

钉钉的接入方式与 Telegram 和飞书不同：

| 特性 | Telegram / 飞书 | 钉钉自定义机器人 |
|------|:---:|:---:|
| 接入层级 | OpenClaw 插件（Gateway 级） | Skill 脚本级 |
| 交互方式 | 双向对话（收发消息） | 单向推送（仅发送） |
| 配置位置 | `openclaw.json` | Skill 的 `.env` 文件 |
| 适用场景 | 与 Agent 对话交互 | 自动推送告警/巡检报告到群 |

**简单来说：**
- Telegram / 飞书 = 用户和 Agent **对话**的渠道
- 钉钉自定义机器人 = Agent 执行 Skill 后，将结果**推送**到钉钉群的通知渠道

## 9.2 创建钉钉自定义机器人

1. 打开钉钉，进入目标群组
2. 群设置 → 智能群助手 → 添加机器人 → 自定义（通过 Webhook 接入）
3. 设置机器人名称和头像
4. **安全设置**选择「加签」，记录：
   - **Webhook URL**：`https://oapi.dingtalk.com/robot/send?access_token=<token>`
   - **签名密钥（Secret）**：`SEC` 开头的字符串

> **推荐使用「加签」模式**，比「关键词」和「IP 白名单」更安全。

## 9.3 在 Skill 中配置

钉钉 Webhook 配置在 Skill 的 `.env` 文件中，而非 `openclaw.json`。

### .env.example 模板

```env
# 钉钉群机器人（--send-dingtalk 时使用）
DINGTALK_WEBHOOK_URL=https://oapi.dingtalk.com/robot/send?access_token=<你的 token>
DINGTALK_SECRET=<你的签名密钥>
```

### 实际 .env 文件

将 `.env.example` 复制为 `.env` 并填入真实值：

```bash
cd ~/.openclaw/workspace/<agent>/skills/<skill-name>/scripts/
cp .env.example .env
# 编辑 .env，填入 Webhook URL 和 Secret
```

> **重要：** `.env` 文件包含凭证信息，不要提交到版本控制。

## 9.4 在 skill.json 中定义钉钉推送命令

在 `skill.json` 的 `commands` 中添加钉钉相关命令：

```json
{
  "commands": [
    {
      "name": "run",
      "description": "执行巡检（仅本地输出，不发送通知）",
      "executable": "bash",
      "arguments": ["scripts/run.sh"]
    },
    {
      "name": "send-dingtalk",
      "description": "执行并发送报告到钉钉群",
      "executable": "bash",
      "arguments": ["scripts/run.sh", "--send-dingtalk"]
    },
    {
      "name": "send-feishu",
      "description": "执行并发送报告到飞书群",
      "executable": "bash",
      "arguments": ["scripts/run.sh", "--send-feishu"]
    },
    {
      "name": "send-all",
      "description": "执行并同时发送到所有通知渠道",
      "executable": "bash",
      "arguments": ["scripts/run.sh", "--send-dingtalk", "--send-feishu"]
    }
  ]
}
```

### 命令设计模式

| 命令 | 说明 |
|------|------|
| `run` | 默认执行，仅本地输出结果 |
| `send-dingtalk` | 执行并推送到钉钉 |
| `send-feishu` | 执行并推送到飞书 |
| `send-all` | 同时推送所有渠道 |
| `--at-all` | 发送时 @所有人（可叠加） |
| `--dry-run` | 预览模式，不实际发送 |

> **重要原则：** Skill 默认不发送通知到任何群。只有用户明确要求"推送到钉钉"、"发到群里"时，Agent 才使用带 `--send-*` 参数的命令。

## 9.5 脚本中实现钉钉推送

### 加签逻辑（Python 示例）

```python
import time
import hmac
import hashlib
import base64
import urllib.parse
import urllib.request
import json
import os

def send_dingtalk(title: str, content: str, at_all: bool = False):
    """发送钉钉 Markdown 消息"""
    webhook_url = os.environ.get("DINGTALK_WEBHOOK_URL")
    secret = os.environ.get("DINGTALK_SECRET")

    if not webhook_url or not secret:
        print("❌ 钉钉配置缺失，跳过推送")
        return

    # 加签
    timestamp = str(round(time.time() * 1000))
    string_to_sign = f"{timestamp}\n{secret}"
    hmac_code = hmac.new(
        secret.encode("utf-8"),
        string_to_sign.encode("utf-8"),
        digestmod=hashlib.sha256,
    ).digest()
    sign = urllib.parse.quote_plus(base64.b64encode(hmac_code))

    url = f"{webhook_url}&timestamp={timestamp}&sign={sign}"

    payload = {
        "msgtype": "markdown",
        "markdown": {"title": title, "text": content},
        "at": {"isAtAll": at_all},
    }

    req = urllib.request.Request(
        url,
        data=json.dumps(payload).encode("utf-8"),
        headers={"Content-Type": "application/json"},
    )
    with urllib.request.urlopen(req, timeout=10) as resp:
        result = json.loads(resp.read())
        if result.get("errcode") != 0:
            print(f"❌ 钉钉发送失败: {result.get('errmsg')}")
        else:
            print("✅ 已推送到钉钉群")
```

### 钉钉 Markdown 格式注意事项

钉钉的 Markdown 支持有限，需注意：

| 支持 | 不支持 |
|------|--------|
| 标题（`#`） | 表格 |
| 粗体（`**`） | 删除线 |
| 链接（`[text](url)`） | 代码块（部分客户端） |
| 有序/无序列表 | 图片嵌入（需用链接） |
| 引用（`>`） | HTML 标签 |

**建议：** 巡检报告使用简洁的列表格式，避免复杂表格。

## 9.6 与定时任务结合

钉钉推送通常配合 Cron 定时任务使用（详见 [第 13 章](./13-cron.md)）：

```
每天 09:00
  → Cron 触发 OpsBot
    → OpsBot 执行 host-monitor 的 send-dingtalk 命令
      → 巡检报告推送到钉钉群
```

### 通过 Agent Manager 设置

```
设置定时任务：每天早上 9 点，让 OpsBot 执行主机巡检并推送钉钉报告
```

### Cron Job 中的 payload 示例

```jsonc
{
  "payload": {
    "kind": "agentTurn",
    "message": "执行主机巡检，使用 send-dingtalk 命令推送结果",
    "timeoutSeconds": 120
  }
}
```

## 9.7 多通知渠道架构

一个 Skill 可以同时支持多个通知渠道：

```
              ┌──────── --send-dingtalk ──── 钉钉群（Markdown 消息）
              │
Skill 执行 ───┼──────── --send-feishu ───── 飞书群（富文本卡片）
              │
              └──────── (默认) ──────────── 仅本地输出（Agent 回复用户）
```

### .env 中的多渠道配置模板

```env
# 钉钉群机器人
DINGTALK_WEBHOOK_URL=https://oapi.dingtalk.com/robot/send?access_token=<token>
DINGTALK_SECRET=<secret>

# 飞书应用（用于主动推送消息到群）
FEISHU_APP_ID=<app_id>
FEISHU_APP_SECRET=<app_secret>
FEISHU_CHAT_ID=<chat_id>
```

## 9.8 安全注意事项

| 风险 | 防护措施 |
|------|----------|
| Webhook URL 泄露 | 存储在 `.env` 中，不提交版本控制 |
| Secret 泄露 | 同上；SKILL.md 安全规则禁止 Agent 透露 |
| 消息注入 | 脚本生成消息内容，不直接转发用户输入 |
| 频率限制 | 钉钉机器人每分钟最多 20 条消息，注意定时任务频率 |

### 在 SKILL.md 中配置安全规则

钉钉相关的安全规则应包含在 Skill 的 SKILL.md 中（详见 [第 11 章](./11-security-rules.md)）：

```markdown
## 绝对禁止

- 不透露 Webhook URL、签名密钥
- 不透露钉钉群 ID、机器人名称
- 不透露通知推送的技术实现方式
```

## 下一步

进入 [第 10 章：为 Agent 创建 Skills](./10-create-skills.md)，把“通知命令模式”系统化复用到更多 Skill。
