# 1. 安装 OpenClaw

## 1.1 系统要求

- macOS (darwin) 或 Linux
- Homebrew（macOS 推荐）
- Node.js 18+（部分功能需要）
- Python 3.x（Skills 脚本需要）

## 1.2 通过 Homebrew 安装（macOS）

```bash
# 添加 OpenClaw tap
brew tap openclaw/tap

# 安装 openclaw-cli
brew install openclaw-cli
```

安装完成后，CLI 路径通常为：

```
/opt/homebrew/Cellar/openclaw-cli/<version>/
```

## 1.3 初始化配置

```bash
# 运行初始化向导
openclaw onboard
```

向导会引导你完成：
1. 创建主配置目录 `~/.openclaw/`
2. 配置 AI 模型提供商和 API Key
3. 创建默认 Agent
4. 初始化 workspace

初始化完成后，目录结构如下：

```
~/.openclaw/
├── openclaw.json          # 主配置文件
├── policy.yaml            # 安全策略文件
├── workspace/             # Agent 工作区目录
├── agents/                # Agent 会话数据
├── skills/                # 全局 Skills 目录
├── credentials/           # 凭证存储
├── identity/              # 设备身份
├── devices/               # 设备配对
├── cron/                  # 定时任务
├── logs/                  # 日志
└── telegram/              # Telegram 状态
```

## 1.4 主配置文件 openclaw.json 结构

```jsonc
{
  "meta": {
    "lastTouchedVersion": "2026.2.9"
  },
  "env": {
    // 环境变量（代理等）
    "HTTPS_PROXY": "http://127.0.0.1:7890",
    "NO_PROXY": "api.moonshot.cn,volces.com"
  },
  "auth": {
    "profiles": {
      "moonshot:default": { "provider": "moonshot", "mode": "api_key" },
      "volcengine:default": { "provider": "volcengine", "mode": "api_key" }
    }
  },
  "models": {
    // 模型提供商配置 → 见下方
  },
  "agents": {
    // Agent 配置 → 第 3/4 章详述
  },
  "bindings": [
    // 渠道绑定 → 第 6/8 章详述
  ],
  "channels": {
    // 通信渠道 → 第 5/7 章详述
  },
  "gateway": {
    // 网关配置
  },
  "plugins": {
    // 插件启用
  }
}
```

## 1.5 配置 AI 模型提供商

在 `openclaw.json` 的 `models` 部分配置：

```jsonc
"models": {
  "mode": "merge",
  "providers": {
    "moonshot": {
      "baseUrl": "https://api.moonshot.cn/v1",
      "apiKey": "<你的 API Key>",
      "api": "openai-completions",
      "models": [
        {
          "id": "kimi-k2.5",
          "name": "Kimi K2.5",
          "reasoning": false,
          "input": ["text"],
          "contextWindow": 256000,
          "maxTokens": 8192
        }
      ]
    },
    "volcengine": {
      "baseUrl": "https://ark.cn-beijing.volces.com/api/coding/v3",
      "apiKey": "<你的 API Key>",
      "api": "openai-completions",
      "models": [
        {
          "id": "ark-code-latest",
          "name": "Coding Plan 默认模型",
          "reasoning": false,
          "input": ["text"],
          "contextWindow": 256000,
          "maxTokens": 8192
        }
      ]
    }
  }
}
```

**支持的模型 API 格式：** `openai-completions`（兼容 OpenAI API 格式的任何提供商）

## 1.6 启动网关

```bash
# 启动 OpenClaw 网关服务
openclaw gateway start
```

网关配置位于 `openclaw.json` 的 `gateway` 部分：

```jsonc
"gateway": {
  "port": 18789,
  "mode": "local",
  "bind": "lan",
  "auth": {
    "mode": "token",
    "token": "<自动生成的访问令牌>"
  },
  "http": {
    "endpoints": {
      "chatCompletions": { "enabled": true }
    }
  }
}
```

## 1.7 验证安装

```bash
# 检查版本
openclaw --version

# 查看当前配置
openclaw config show

# 列出所有 Agent
openclaw agents list
```
