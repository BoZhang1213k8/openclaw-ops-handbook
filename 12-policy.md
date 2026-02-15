# 12. Policy 权限策略配置

## 12.1 概述

`policy.yaml` 是 OpenClaw 的系统级安全策略文件，控制三大类权限：

1. **工具权限** — 哪些 Agent 可以使用哪些工具
2. **文件权限** — 哪些 Agent 可以读写哪些文件
3. **技能权限** — 哪些 Agent 可以使用哪些 Skill

文件位置：`~/.openclaw/policy.yaml`

## 12.2 工具权限

```yaml
tools:
  # 网关配置权限 — 仅 Agent Manager
  gateway:
    deny: ["*"]
    allow: ["agent-manager"]

  # 命令执行 — 所有 Agent
  exec:
    allow: ["*"]

  # 定时任务 — 仅管理和运维
  cron:
    deny: ["*"]
    allow: ["agent-manager", "ops"]

  # Web 工具 — 仅 Agent Manager
  web_search:
    deny: ["*"]
    allow: ["agent-manager"]
  web_fetch:
    deny: ["*"]
    allow: ["agent-manager"]
  browser:
    deny: ["*"]
    allow: ["agent-manager"]
  tts:
    deny: ["*"]
    allow: ["agent-manager"]
```

### 可用工具列表

| 工具 | 说明 | 建议分配 |
|------|------|----------|
| `gateway` | 修改网关配置 | 仅 agent-manager |
| `exec` | 执行系统命令 | 所有 Agent（Skill 执行需要） |
| `cron` | 管理定时任务 | agent-manager + 特定 Agent |
| `web_search` | 网络搜索 | 仅 agent-manager |
| `web_fetch` | 获取网页内容 | 仅 agent-manager |
| `browser` | 浏览器自动化 | 仅 agent-manager |
| `tts` | 文字转语音 | 按需 |

## 12.3 文件权限

### 基本结构

```yaml
files:
  read:
    allow:
      - "<允许读取的路径>"
    deny:
      - "<禁止读取的路径>"

  write:
    allow:
      - "<允许写入的路径>"
    deny:
      - "<禁止写入的路径>"
```

### 推荐配置

```yaml
files:
  # 读取权限
  read:
    allow:
      # 允许读取 workspace（各 Agent 只能看到自己的）
      - "~/.openclaw/workspace/**"
    deny:
      # 禁止读取系统配置
      - "~/.openclaw/openclaw.json"
      - "~/.openclaw/policy.yaml"
      # 禁止读取全局 Skills 目录
      - "/opt/homebrew/Cellar/openclaw-cli/**/skills/**"
      - "~/.openclaw/skills/**"

  # 写入权限
  write:
    allow:
      - "~/.openclaw/workspace/**"
    deny:
      - "/**"    # 默认禁止写入其他位置
```

### Agent 特定权限

可以为特定 Agent 设置独立的文件权限：

```yaml
files:
  # Agent Manager 特殊权限
  agent-manager:
    allow:
      - "~/.openclaw/workspace/agent-manager/**"
      - "~/.openclaw/skills/**"                      # 允许读全局 Skills
    deny:
      - "~/.openclaw/workspace/ops-agent/**"          # 禁止访问其他 Agent
      - "~/.openclaw/workspace/broadband-ops-agent/**"
      - "~/.openclaw/openclaw.json"
      - "~/.openclaw/policy.yaml"

  # BroadbandOpsBot 特殊权限
  broadband-ops:
    allow:
      - "~/.openclaw/workspace/broadband-ops-agent/**"
      - "/path/to/external/project/**"               # 允许访问外部项目
    deny:
      - "~/.openclaw/workspace/agent-manager/**"
      - "~/.openclaw/workspace/ops-agent/**"
      - "~/.openclaw/openclaw.json"
      - "~/.openclaw/policy.yaml"
      - "~/.openclaw/skills/**"
```

### 权限优先级

```
Agent 特定 deny > Agent 特定 allow > 全局 deny > 全局 allow
```

## 12.4 技能权限

### 基本结构

```yaml
skills:
  default: deny          # 默认拒绝所有 Skills

  <agent-id>:
    default: allow/deny  # 该 Agent 的默认策略
    allow:
      - "<skill 匹配模式>"
    deny:
      - "<skill 匹配模式>"
```

### Skill 匹配模式

```
<agent-id>:workspace/<skill-name>   — workspace 下的 Skill
<skill-name>                         — 全局 Skill
*                                    — 所有 Skills
```

### 推荐配置

```yaml
skills:
  # 全局默认：拒绝
  default: deny

  # Agent Manager — 允许使用全局 Skills 和自身 Skills
  agent-manager:
    default: allow
    deny:
      - "ops:workspace/*"              # 不能用 OpsBot 的 Skills
      - "broadband-ops:workspace/*"    # 不能用 BroadbandOpsBot 的 Skills

  # BroadbandOpsBot — 只能用自己的 Skills
  broadband-ops:
    allow:
      - "broadband-ops:workspace/*"    # 只允许自己 workspace 下的
    deny:
      - "ops:workspace/*"
      - "agent-manager:workspace/*"

  # OpsBot — 只能用自己的 Skills
  ops:
    allow:
      - "ops:workspace/*"
    deny:
      - "broadband-ops:workspace/*"
      - "agent-manager:workspace/*"
```

## 12.5 完整 policy.yaml 示例

```yaml
# Policy 配置 - 绝对隔离各 Agent 权限

# 工具权限控制
tools:
  gateway:
    deny: ["*"]
    allow: ["agent-manager"]
  exec:
    allow: ["*"]
  cron:
    deny: ["*"]
    allow: ["agent-manager", "ops"]
  web_search:
    deny: ["*"]
    allow: ["agent-manager"]
  web_fetch:
    deny: ["*"]
    allow: ["agent-manager"]
  browser:
    deny: ["*"]
    allow: ["agent-manager"]

# 文件访问控制
files:
  read:
    allow:
      - "~/.openclaw/workspace/**"
    deny:
      - "~/.openclaw/openclaw.json"
      - "~/.openclaw/policy.yaml"
      - "~/.openclaw/skills/**"
  write:
    allow:
      - "~/.openclaw/workspace/**"
    deny:
      - "/**"

  # Agent 特定权限
  agent-manager:
    allow:
      - "~/.openclaw/workspace/agent-manager/**"
      - "~/.openclaw/skills/**"
    deny:
      - "~/.openclaw/workspace/ops-agent/**"
      - "~/.openclaw/workspace/broadband-ops-agent/**"
      - "~/.openclaw/openclaw.json"
      - "~/.openclaw/policy.yaml"

  broadband-ops:
    allow:
      - "~/.openclaw/workspace/broadband-ops-agent/**"
    deny:
      - "~/.openclaw/workspace/agent-manager/**"
      - "~/.openclaw/workspace/ops-agent/**"
      - "~/.openclaw/openclaw.json"
      - "~/.openclaw/policy.yaml"
      - "~/.openclaw/skills/**"

# 技能访问控制
skills:
  default: deny
  agent-manager:
    default: allow
    deny:
      - "ops:workspace/*"
      - "broadband-ops:workspace/*"
  broadband-ops:
    allow:
      - "broadband-ops:workspace/*"
    deny:
      - "ops:workspace/*"
      - "agent-manager:workspace/*"
```

## 12.6 新增 Agent 时的 Policy 更新

每次创建新 Agent，需要在 `policy.yaml` 中更新以下内容：

1. **files** 部分：为新 Agent 添加独立的文件权限规则
2. **skills** 部分：为新 Agent 添加 Skill 访问规则
3. **其他 Agent 的 deny 列表**：添加新 Agent 的 workspace 路径，防止互相访问
4. **tools** 部分：按需分配工具权限

### 示例：新增一个 `data-analyst` Agent

```yaml
# 1. files 部分 — 添加新 Agent 的独立权限
files:
  data-analyst:
    allow:
      - "~/.openclaw/workspace/data-analyst-agent/**"
    deny:
      - "~/.openclaw/workspace/agent-manager/**"
      - "~/.openclaw/workspace/ops-agent/**"
      - "~/.openclaw/workspace/broadband-ops-agent/**"   # 禁止访问其他 Agent
      - "~/.openclaw/openclaw.json"
      - "~/.openclaw/policy.yaml"
      - "~/.openclaw/skills/**"

# 2. skills 部分 — 添加新 Agent 的 Skill 权限
skills:
  data-analyst:
    allow:
      - "data-analyst:workspace/*"     # 只能用自己的 Skills
    deny:
      - "ops:workspace/*"
      - "broadband-ops:workspace/*"
      - "agent-manager:workspace/*"

# 3. 同时更新现有 Agent 的 deny 列表，加上新 Agent 的 workspace
# 例如在 broadband-ops 的 deny 中添加：
#   - "~/.openclaw/workspace/data-analyst-agent/**"
# 在 agent-manager 的 deny 中添加：
#   - "~/.openclaw/workspace/data-analyst-agent/**"
```
