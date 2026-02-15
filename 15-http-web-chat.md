# 15. 通过 HTTP API 和 Web 聊天界面访问 Agent

## 15.1 概述

除了 Telegram 和飞书，OpenClaw 还支持通过 HTTP API 访问 Agent。基于此能力，可以搭建一个局域网 Web 聊天服务器，让非技术同事打开浏览器即可与 Agent 对话，无需安装任何软件。

```
┌──────────────┐     HTTP      ┌──────────────────┐    HTTP API    ┌──────────────┐
│ 同事浏览器    │ ──────────▶  │ LAN Chat Server  │ ────────────▶ │ OpenClaw     │
│ (任意设备)    │  :28765      │ (Python, 本机)    │  :18789       │ Gateway      │
└──────────────┘              └──────────────────┘               └──────────────┘
                                     │                                   │
                               IP 白名单访问控制              Bearer Token 认证
                               用户端无需密码                 服务端安全代理
```

**两个访问路径：**

| 路径 | 说明 | 使用场景 |
|------|------|----------|
| `/broadband` | 单 Agent 专属页（宽带运维） | 给同事使用的简洁界面 |
| `/claw-x9f7` | 多 Agent 切换页 | 可在多个 Agent 之间切换 |

## 15.2 前置条件：开启 Gateway HTTP API

在 `openclaw.json` 中确保 Chat Completions API 已启用（Gateway 基础配置参见 [第 1 章](./01-install.md#16-启动网关)）：

```jsonc
"gateway": {
  "port": 18789,
  "mode": "local",
  "bind": "lan",
  "auth": {
    "mode": "token",
    "token": "<自动生成的 Token>"
  },
  "http": {
    "endpoints": {
      "chatCompletions": {
        "enabled": true          // 必须为 true
      }
    }
  }
}
```

### Gateway HTTP API 格式

OpenClaw Gateway 兼容 OpenAI Chat Completions API 格式：

**请求：**

```
POST /v1/chat/completions
Authorization: Bearer <gateway-token>
Content-Type: application/json
x-openclaw-agent-id: <agent-id>        # 指定目标 Agent
```

```json
{
  "model": "openclaw",
  "messages": [
    {"role": "user", "content": "你好"}
  ]
}
```

**响应：**

```json
{
  "choices": [
    {
      "message": {
        "role": "assistant",
        "content": "你好！有什么可以帮你的？"
      }
    }
  ]
}
```

**关键 Header：** `x-openclaw-agent-id` 用于路由到指定 Agent（如 `broadband-ops`、`ops`）。

## 15.3 LAN Chat Server 架构

LAN Chat Server 是一个纯 Python 实现的轻量级 HTTP 服务器，充当 Gateway 的代理层：

```
功能特性：
├── Web 界面
│   ├── /broadband    — 单 Agent 专属页（宽带运维）
│   └── /claw-x9f7    — 多 Agent 切换页
├── API 代理
│   └── /api/chat     — 前端调用 → 代理到 Gateway /v1/chat/completions
├── 安全
│   ├── IP 白名单     — 只允许指定 IP/网段访问
│   └── Token 隐藏    — Gateway Token 在服务端，用户端不可见
├── 会话管理
│   ├── 独立会话       — 每个浏览器 + 每个 Agent 独立对话历史
│   ├── 历史记录       — 保留最近 50 轮对话
│   └── 自动过期       — 4 小时无活动自动清理
└── 生命周期
    ├── 后台守护进程    — start/stop/restart/status
    └── 日志文件       — 记录访问和错误
```

## 15.4 部署 LAN Chat Server

### Skill 结构

```
lan-chat-server/
├── skill.json              # 技能元数据
├── SKILL.md                # Agent 使用指南
└── scripts/
    ├── run.sh              # 入口脚本
    ├── chat_server.py      # 主程序（内嵌 HTML + 代理服务）
    ├── ip_whitelist.conf   # IP 白名单
    ├── .env.example        # 环境变量模板
    ├── .env                # 实际环境变量（不提交版本控制）
    ├── .chat-server.pid    # PID 文件（运行时生成）
    └── .chat-server.log    # 运行日志（自动生成）
```

### 步骤一：配置环境变量

复制 `.env.example` 为 `.env` 并填写：

```env
# 聊天服务器监听端口
CHAT_SERVER_PORT=28765

# OpenClaw Gateway 地址（通常是本机）
OPENCLAW_GATEWAY_URL=http://127.0.0.1:18789

# OpenClaw Gateway Token（从 openclaw.json 的 gateway.auth.token 获取，参见第 1 章）
OPENCLAW_GATEWAY_TOKEN=<你的 gateway token>
```

### 步骤二：配置 IP 白名单

编辑 `scripts/ip_whitelist.conf`：

```conf
# IP 白名单 - 每行一条规则
# 支持三种格式：

# 精确 IP
127.0.0.1

# 网段前缀（匹配所有以此开头的 IP）
10.xxx.xxx.

# CIDR 格式
192.168.1.0/24
```

**特性：**
- 修改白名单后**实时生效**，无需重启服务
- 如果白名单文件为空或不存在，则放行所有 IP
- 空行和 `#` 开头的注释行会被忽略

### 步骤三：启动服务

通过 Agent Manager 管理（推荐）：

```
# 在 Agent Manager 的群组中发送
启动聊天服务器
```

或通过命令行：

```bash
# 启动（后台守护进程）
bash scripts/run.sh start

# 自定义端口
bash scripts/run.sh start --port 9090

# 查看状态
bash scripts/run.sh status

# 停止
bash scripts/run.sh stop

# 重启
bash scripts/run.sh restart
```

### 步骤四：验证

```bash
# 查看状态
bash scripts/run.sh status
# 输出: [chat-server] ✅ 运行中 (PID=12345, port=28765)
#       [chat-server] 📎 访问地址: http://<本机IP>:28765
```

在浏览器中访问：
- `http://<服务器IP>:28765/broadband` — 宽带运维专属页
- `http://<服务器IP>:28765/claw-x9f7` — 多 Agent 切换页

## 15.5 Web 界面功能

### 单 Agent 专属页（/broadband）

为特定业务场景定制的简洁界面：
- 固定绑定一个 Agent（如宽带运维）
- 页面标题、配色、欢迎语均可定制
- 适合分享给非技术同事

### 多 Agent 切换页（/claw-x9f7）

支持在多个 Agent 之间切换：
- 顶部 Agent 选择栏
- 每个 Agent 独立的对话历史
- 切换 Agent 时对话上下文自动保存和恢复

### 共通功能

| 功能 | 说明 |
|------|------|
| Markdown 渲染 | 自动渲染 Agent 回复中的 Markdown（表格、代码块、列表等） |
| 对话历史 | 浏览器端 localStorage 持久化 + 服务端保留最近 50 轮 |
| 右键菜单 | 复制文本 / 复制 Markdown / 复制富文本 / 复制全部对话 |
| 特殊命令 | `/clear`、`/reset`、`清空对话`、`重新开始` |
| 自动滚动 | 新消息自动滚动到底部 |
| 响应式布局 | 适配桌面和移动设备 |

## 15.6 添加新的 Agent 专属页

如需为其他 Agent 创建专属入口页面，在 `chat_server.py` 中添加：

### 1. 生成专属页面

```python
# 在文件中使用 make_single_agent_page 函数生成
MY_AGENT_PAGE = make_single_agent_page(
    agent_id="my-agent",              # Agent ID
    agent_name="我的助手",             # 页面显示名称
    agent_emoji="🤖",                 # Agent Emoji
    title="我的助手",                  # 浏览器标题
    accent_from="#6366f1",            # 渐变色起点
    accent_to="#8b5cf6",              # 渐变色终点
)
```

### 2. 注册路由

```python
# 在 ChatHandler.do_GET 方法中添加路由
def do_GET(self):
    if not self._check_access():
        return
    if self.path in ('/broadband', '/broadband/'):
        self._serve_html(BROADBAND_PAGE)
    elif self.path in ('/my-agent', '/my-agent/'):      # 新路由
        self._serve_html(MY_AGENT_PAGE)
    elif self.path in ('/claw-x9f7', '/claw-x9f7/'):
        self._serve_html(HTML_PAGE)
    else:
        self.send_response(404)
        ...
```

### 3. 重启服务

```bash
bash scripts/run.sh restart
```

现在可以通过 `http://<IP>:28765/my-agent` 访问新的专属页面。

## 15.7 在多 Agent 切换页中添加 Agent

如需在 `/claw-x9f7` 多 Agent 页面中添加新的 Agent 选项，修改 `HTML_PAGE` 中的两处：

### 1. 添加切换按钮

在 `agent-selector` div 中添加：

```html
<div class="agent-selector">
  <button class="agent-btn active" data-agent="broadband-ops">🌐 宽带运维</button>
  <button class="agent-btn" data-agent="ops">🖥️ OpsBot</button>
  <button class="agent-btn" data-agent="my-agent">🤖 我的助手</button>  <!-- 新增 -->
</div>
```

### 2. 注册 Agent 名称映射

在 JavaScript 中添加：

```javascript
const agentNames = {
  'broadband-ops': '🌐 宽带运维助手',
  'ops': '🖥️ OpsBot',
  'my-agent': '🤖 我的助手'   // 新增
};
```

## 15.8 安全注意事项

### Token 安全

- Gateway Token 保存在服务端的 `.env` 文件中
- 用户端浏览器**不接触 Token**，所有 API 调用由服务端代理
- `.env` 文件不提交到版本控制

### IP 白名单

- 生产环境**必须**配置 IP 白名单
- 只允许办公网段访问，阻止外部访问
- 白名单支持三种格式：精确 IP、网段前缀、CIDR
- 修改即时生效，无需重启

### 路由隐蔽

- 对外分享的专属页面路径可以是简洁的业务名称（如 `/broadband`）
- 管理员使用的多 Agent 页面可以使用不易猜测的路径（如 `/claw-x9f7`）

### 会话隔离

- 每个浏览器 session + 每个 Agent 独立维护对话历史
- 不同用户之间对话完全隔离
- 浏览器端对话记录存储在内存中，刷新或关闭页面即清除

### 会话回收

OpenClaw 核心 Gateway 不会自动回收过期会话，需要通过 Skill 定期清理。

已部署 `session-cleanup` Skill（位于 Agent Manager 的 workspace skills 下）：

**两种维护策略：**

| 维护类型 | 适用对象 | 触发条件 |
|----------|----------|----------|
| TTL 过期清理 | HTTP API / Web / Cron 临时会话 | 4 小时无活动 |
| 满载重置 | **所有会话**（含 Telegram / 飞书 / 主控台） | 上下文使用率 ≥80% |

- **执行频率**：系统 crontab 每小时整点自动执行（`--auto` 模式）
- **安全措施**：每次操作前自动备份（.jsonl 备份为 .bak-before-reset）
- **满载重置**：渠道会话重置后，下次对话自动创建新会话，不影响渠道绑定

**通过 Agent Manager 使用（推荐）：**

在 Agent Manager 的群组中直接下达指令即可：

```
查看当前会话状态
清理过期会话
哪些会话快满了
重置满载的会话
```

Agent Manager 会自动调用 `session-cleanup` Skill 的对应命令。

**Skill 提供的命令：**

| 命令 | 说明 |
|------|------|
| `status` | 查看各 Agent 会话数量、磁盘占用和满载告警 |
| `dry-run` | 预览将要清理的过期临时会话（不删除） |
| `cleanup` | 执行清理过期临时会话（默认 TTL 4 小时） |
| `cleanup-ttl` | 自定义 TTL 清理（参数：小时数） |
| `cleanup-agent` | 只清理指定 Agent（参数：agent ID） |
| `overloaded` | 列出上下文 ≥80% 的满载会话（参数：阈值百分比） |
| `reset-overloaded` | 重置满载会话，包括受保护的渠道会话 |
| `reset-overloaded-dry` | 预览满载重置（不实际执行） |
| `auto` | 自动维护：清理过期 + 重置满载（crontab 调用） |
| `logs` | 查看自动维护历史日志 |

## 15.9 skill.json 配置

```json
{
  "name": "lan-chat-server",
  "version": "1.0.0",
  "description": "局域网 Web 聊天服务器",
  "author": "Alan Zhang",
  "tags": ["chat", "webchat", "lan", "web-ui", "proxy", "browser"],
  "requirements": {
    "platforms": ["darwin", "linux"],
    "dependencies": ["python3"]
  },
  "commands": [
    {
      "name": "start",
      "description": "启动聊天服务器（后台守护进程）",
      "executable": "bash",
      "arguments": ["scripts/run.sh", "start"]
    },
    {
      "name": "stop",
      "description": "停止聊天服务器",
      "executable": "bash",
      "arguments": ["scripts/run.sh", "stop"]
    },
    {
      "name": "restart",
      "description": "重启聊天服务器",
      "executable": "bash",
      "arguments": ["scripts/run.sh", "restart"]
    },
    {
      "name": "status",
      "description": "查看聊天服务器运行状态",
      "executable": "bash",
      "arguments": ["scripts/run.sh", "status"]
    }
  ]
}
```

## 15.10 与其他渠道的对比

| 特性 | Telegram | 飞书 | Web Chat (HTTP API) |
|------|:---:|:---:|:---:|
| 需要安装软件 | Telegram App | 飞书 App | 浏览器即可 |
| 需要注册账号 | 需要 | 需要（企业） | 不需要 |
| 适合非技术用户 | 一般 | 较好 | 最佳 |
| 局域网部署 | 需代理 | 需回调 | 直接可用 |
| 多 Agent 切换 | 通过群组区分 | 通过群组区分 | 页面内切换 |
| 对话历史 | 永久保存 | 永久保存 | 浏览器端 + 服务端临时 |
| 通知推送 | 支持 | 支持 | 不支持（需主动访问） |
| 集成定时任务 | 支持 | 支持 | 不支持 |
| 安全管控 | Bot Token + 白名单 | App 密钥 + 权限 | IP 白名单 + Token 代理 |
