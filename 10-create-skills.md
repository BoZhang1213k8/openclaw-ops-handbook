# 10. 为 Agent 创建 Skills

## 10.0 在 Cursor 中基于 Plan 创建 Skill（推荐）

本手册中的 broadband-ops-agent 相关 Skill 均在 Cursor 中创建，并统一使用标准 Plan 作为模板，确保目录结构、文件规范、符号链接和 AGENTS.md/TOOLS.md 注册流程一致。

### Plan 文件位置

```
~/.cursor/plans/broadband-ops-agent 新建 Skill 标准流程.plan.md
```

### 操作步骤

**步骤 1：打开 Cursor，进入 openClawSkills 项目**

确保工作区为包含 `openClawSkills/` 目录的项目。

**步骤 2：引用 Plan 并描述需求**

在 Cursor 的 AI 对话中输入，引用 Plan 并说明要创建的 Skill：

```
@~/.cursor/plans/broadband-ops-agent 新建 Skill 标准流程.plan.md 参考这个 plan，新建一个 skill：
- 名称：{skill-name}
- 用途：{业务描述}
- 是否需要用户参数：是/否
- 是否需要 dry-run 预览：是/否
- 是否需要用户确认后再执行：是/否
- 其他特殊要求：（如有）
```

**步骤 3：确认 Plan 中的 Todo 并执行**

Plan 包含 8 个 Todo 步骤，AI 会按序执行：

1. 确定 skill 名称、用途、参数需求、执行流程
2. 在 openClawSkills/ 下创建目录结构
3. 编写 skill.json
4. 编写 SKILL.md
5. 编写 scripts/（run.sh、主逻辑脚本、config.py、.env.*）
6. 创建符号链接到 broadband-ops-agent/skills/
7. 更新 AGENTS.md
8. 更新 TOOLS.md

**步骤 4：提供参考 Skill（可选）**

若新 Skill 与现有 Skill 类似，可同时引用参考样例，例如：

```
参考 @order-archive 的流程设计
参考 @mbhopen-delete 的参数化 SQL 写法
```

### 示例：创建 mbhopen-delete 类 Skill

```
@~/.cursor/plans/broadband-ops-agent 新建 Skill 标准流程.plan.md 参考这个 plan，新建一个 skill：
DELETE FROM WO_MBHOPEN_RUSULT WHERE SN = :sn;
用户需要提供 SN 参数，流程参考 order-archive（先 dry-run 预览，等待用户确认后再执行）
```

### 注意事项

- **开发目录**：Skill 源码创建在 `~/openClawSkills/{skill-name}/`，与 Agent 工作区通过符号链接关联
- **Plan 不修改**：执行时按 Plan 的 Todo 实施，不要修改 Plan 文件本身
- **检查清单**：完成后对照 Plan 末尾的检查清单逐项确认

---

## 10.1 Skill 概述

Skill 是 Agent 的专业能力单元。每个 Skill 包含：

```
skills/<skill-name>/
├── skill.json        # 技能元数据、命令定义
├── SKILL.md          # Agent 使用指南（含安全规则）
└── scripts/          # 执行脚本
    ├── run.sh        # 入口脚本（Bash）
    ├── main.py       # 核心逻辑（Python）
    ├── config.py     # 配置文件
    ├── .env          # 环境变量（生产）
    ├── .env.test     # 环境变量（测试）
    └── .env.example  # 环境变量模板
```

## 10.2 Skill 存放位置

Skills 可以放在两个位置：

| 位置 | 说明 | 路径 |
|------|------|------|
| 全局 Skills | 所有 Agent 可用（受 policy 限制） | `~/.openclaw/skills/<skill-name>/` |
| 工作区 Skills | 仅所属 Agent 可用 | `~/.openclaw/workspace/<agent>/skills/<skill-name>/` |

**推荐：** 使用工作区 Skills，符合隔离原则。

## 10.3 创建 skill.json

`skill.json` 定义 Skill 的元数据和可执行命令：

```json
{
  "name": "my-skill",
  "version": "1.0.0",
  "description": "技能描述",
  "author": "Your Name",
  "tags": ["tag1", "tag2"],
  "requirements": {
    "platforms": ["darwin", "linux"],
    "dependencies": ["python3", "bash"]
  },
  "commands": [
    {
      "name": "execute",
      "description": "执行操作",
      "executable": "bash",
      "arguments": ["scripts/run.sh", "--env", "test"]
    },
    {
      "name": "dry-run",
      "description": "预览模式（不实际执行）",
      "executable": "bash",
      "arguments": ["scripts/run.sh", "--env", "test", "--dry-run"]
    }
  ]
}
```

### 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `name` | string | 技能唯一名称 |
| `version` | string | 版本号（语义化版本） |
| `description` | string | 技能简要描述 |
| `author` | string | 作者 |
| `tags` | string[] | 标签，用于搜索和分类 |
| `requirements.platforms` | string[] | 支持的平台 |
| `requirements.dependencies` | string[] | 依赖项 |
| `commands` | Command[] | 可执行命令列表 |

### Command 字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `name` | string | 命令名称 |
| `description` | string | 命令描述（Agent 看到的） |
| `executable` | string | 执行器（`bash`、`python3`、`osascript`） |
| `arguments` | string[] | 命令参数 |

## 10.4 命令设计模式

### 模式一：环境切换

```json
{
  "commands": [
    {
      "name": "execute",
      "description": "在测试环境执行",
      "executable": "bash",
      "arguments": ["scripts/run.sh", "--env", "test"]
    },
    {
      "name": "execute-prod",
      "description": "在生产环境执行",
      "executable": "bash",
      "arguments": ["scripts/run.sh", "--env", "prod"]
    }
  ]
}
```

### 模式二：预览 + 执行

```json
{
  "commands": [
    {
      "name": "dry-run",
      "description": "预览将要处理的记录（不实际修改）",
      "executable": "bash",
      "arguments": ["scripts/run.sh", "--env", "test", "--dry-run"]
    },
    {
      "name": "execute",
      "description": "实际执行操作",
      "executable": "bash",
      "arguments": ["scripts/run.sh", "--env", "test"]
    }
  ]
}
```

### 模式三：多查询方式

```json
{
  "commands": [
    {
      "name": "query-by-id",
      "description": "按 ID 查询",
      "executable": "bash",
      "arguments": ["scripts/run.sh", "--env", "test", "--dry-run", "--id"]
    },
    {
      "name": "query-by-name",
      "description": "按名称查询",
      "executable": "bash",
      "arguments": ["scripts/run.sh", "--env", "test", "--dry-run", "--name"]
    }
  ]
}
```

### 模式四：服务生命周期

```json
{
  "commands": [
    {
      "name": "start",
      "description": "启动服务",
      "executable": "bash",
      "arguments": ["scripts/run.sh", "start"]
    },
    {
      "name": "stop",
      "description": "停止服务",
      "executable": "bash",
      "arguments": ["scripts/run.sh", "stop"]
    },
    {
      "name": "status",
      "description": "查看服务状态",
      "executable": "bash",
      "arguments": ["scripts/run.sh", "status"]
    }
  ]
}
```

### 模式五：多通知渠道

```json
{
  "commands": [
    {
      "name": "run",
      "description": "执行巡检",
      "executable": "bash",
      "arguments": ["scripts/run.sh"]
    },
    {
      "name": "send-dingtalk",
      "description": "执行并发送钉钉通知",
      "executable": "bash",
      "arguments": ["scripts/run.sh", "--send-dingtalk"]
    },
    {
      "name": "send-feishu",
      "description": "执行并发送飞书通知",
      "executable": "bash",
      "arguments": ["scripts/run.sh", "--send-feishu"]
    }
  ]
}
```

## 10.5 创建 SKILL.md

`SKILL.md` 是 Agent 的使用指南，告诉 Agent 何时、如何使用该技能。

### 基本结构

```markdown
---
name: my-skill
description: 当用户需要 xxx 时使用此技能
metadata:
  openclaw:
    emoji: "🔧"
    agents: ["my-agent"]
---

## 使用场景

当用户提到以下需求时使用此技能：
- 需求 1
- 需求 2

## 使用方法

1. 先使用 `dry-run` 命令预览
2. 确认后使用 `execute` 命令执行
3. 向用户报告结果

## 注意事项

- 生产环境操作需用户确认
- 结果展示时注意数据脱敏
```

> **安全规则部分详见 [第 11 章](./11-security-rules.md)**

## 10.6 创建入口脚本 run.sh

```bash
#!/bin/bash
# 入口脚本 - 管理环境和依赖

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"

# 加载环境变量
ENV="test"
for arg in "$@"; do
  if [[ "$arg" == "--env" ]]; then
    shift
    ENV="$1"
    shift
    break
  fi
done

ENV_FILE="${SCRIPT_DIR}/.env.${ENV}"
if [[ -f "$ENV_FILE" ]]; then
  set -a
  source "$ENV_FILE"
  set +a
else
  echo "❌ 环境配置文件不存在: ${ENV}"
  exit 1
fi

# 安装依赖（如需要）
if ! python3 -c "import oracledb" 2>/dev/null; then
  echo "正在安装运行依赖..."
  pip3 install oracledb --quiet
fi

# 执行 Python 脚本
python3 "${SCRIPT_DIR}/main.py" "$@"
```

## 10.7 环境变量配置

### .env.example（模板，提交到版本控制）

```env
# 数据库配置
DB_HOST=
DB_PORT=
DB_USER=
DB_PASSWORD=
DB_SERVICE=
```

### .env.test / .env.prod（实际配置，不提交到版本控制）

```env
DB_HOST=10.xxx.xxx.xxx
DB_PORT=1521
DB_USER=myuser
DB_PASSWORD=mypassword
DB_SERVICE=myservice
```

> **重要：** `.env` 文件包含敏感信息，不要提交到版本控制。在 `.gitignore` 中添加 `.env*`（保留 `.env.example`）。

## 10.8 使用符号链接管理 Skills

如果 Skill 源码在外部项目中维护，可以使用符号链接：

```bash
# 删除 workspace 中的副本
rm -rf ~/.openclaw/workspace/broadband-ops-agent/skills/pboss-resend

# 创建符号链接指向源码
ln -s /path/to/your/project/pboss-resend \
      ~/.openclaw/workspace/broadband-ops-agent/skills/pboss-resend
```

**优点：**
- 只维护一份源码
- 用 IDE 编辑源码，Agent 自动获取更新
- 源码可以纳入版本控制

## 10.9 在 TOOLS.md 中注册

创建或更新 Agent 工作区下的 `TOOLS.md`（路径：`~/.openclaw/workspace/<workspace-name>/TOOLS.md`），添加新 Skill 的说明，让 Agent 知道何时和如何使用它：

```markdown
## 可用 Skills

- `my-skill` — 技能描述
  - 参数：`--env test|prod`、`--dry-run`
  - 使用方法：先 dry-run 预览，再 execute 执行
```

## 10.10 实际示例

### 示例：PBOSS 重新回单技能

```json
{
  "name": "pboss-resend",
  "version": "1.2.0",
  "description": "重新回单给 PBOSS 或二编系统",
  "author": "Alan",
  "tags": ["运维", "回单", "业务号码", "定单编码"],
  "requirements": {
    "platforms": ["darwin", "linux"],
    "dependencies": ["python3", "bash"]
  },
  "commands": [
    {
      "name": "resend-by-acc-nbr",
      "description": "按业务号码查询并重新回单",
      "executable": "bash",
      "arguments": ["scripts/run.sh", "--env", "test", "--acc-nbr"]
    },
    {
      "name": "dry-run-by-acc-nbr",
      "description": "按业务号码预览（不执行）",
      "executable": "bash",
      "arguments": ["scripts/run.sh", "--env", "test", "--dry-run", "--acc-nbr"]
    }
  ]
}
```

### 示例：主机监控技能

```json
{
  "name": "host-monitor",
  "version": "1.0.0",
  "description": "主机资源监控自动巡检",
  "author": "Alan",
  "tags": ["监控", "巡检", "主机"],
  "requirements": {
    "platforms": ["darwin", "linux"],
    "dependencies": ["python3", "bash"]
  },
  "commands": [
    {
      "name": "run",
      "description": "执行主机巡检",
      "executable": "bash",
      "arguments": ["scripts/run_monitor.sh"]
    },
    {
      "name": "send-dingtalk",
      "description": "巡检并推送钉钉通知",
      "executable": "bash",
      "arguments": ["scripts/run_monitor.sh", "--send-dingtalk", "--at-all"]
    }
  ]
}
```

## 10.11 broadband-ops-agent 新建 Skill 标准流程（Plan 内容参考）

本节内容与 Cursor Plan `~/.cursor/plans/broadband-ops-agent 新建 Skill 标准流程.plan.md` 一致，供手动创建或查阅时参考。**推荐使用 [10.0 在 Cursor 中基于 Plan 创建](#100-在-cursor-中基于-plan-创建-skill推荐)** 的方式创建新 Skill。

本节为在 broadband-ops-agent 中新建 skill 的通用流程。执行前需先确定 skill 的具体需求（名称、用途、是否需要参数、是否有 dry-run、是否需要用户确认等）。

### 路径约定

| 用途 | 路径 |
|------|------|
| 开发目录（源码） | `~/openClawSkills/{skill-name}/` |
| Agent 工作区 | `~/.openclaw/workspace/broadband-ops-agent/` |
| Skill 挂载点 | `broadband-ops-agent/skills/{skill-name}` → 符号链接到 openClawSkills |

### 标准目录结构

```
openClawSkills/{skill-name}/
├── skill.json          # 技能元数据、commands 定义
├── SKILL.md            # Agent 操作指引 + 安全保密规则
└── scripts/
    ├── run.sh          # Bash 入口：环境检测、依赖、加载 .env、调用主脚本
    ├── <主逻辑>.py     # 或 .sh，实现业务逻辑
    ├── config.py       # 数据库/API 等配置（若需要）
    ├── .env.example    # 环境变量模板
    ├── .env.test       # 测试环境
    └── .env.prod       # 生产环境
```

**参考样例：** `order-archive`、`mbhopen-delete`、`ticket-restart`、`pboss-resend`、`file-resend`

### 各文件设计要点

**skill.json：** `name`、`version`、`description`、`tags`、`requirements.platforms`、`requirements.dependencies`、`commands`（每项含 `name`、`description`、`executable`、`arguments`）

**SKILL.md：** YAML frontmatter（`name`、`description`、`metadata.openclaw.emoji`、`metadata.openclaw.agents: ["broadband-ops"]`）；安全保密规则；操作指引（Agent Internal）

**run.sh：** 检查 python3 及依赖；解析 `--env` 参数（默认 test）；加载 `.env.{ENV_NAME}`；调用主脚本

**主逻辑脚本：** 参数化查询、禁止 SQL 注入、输出业务结果不暴露技术细节

**环境配置：** 若需数据库则 `config.py` + `.env.*`，复用 order-archive 的 Oracle 连接模式

### 部署：创建符号链接

```bash
ln -s ~/openClawSkills/{skill-name} \
      ~/.openclaw/workspace/broadband-ops-agent/skills/{skill-name}
```

### 更新 Agent 角色配置

**AGENTS.md 三处：**

1. 工作范围 > 主要职责：`- **{业务描述}**（通过 {skill-name} skill）`
2. 状态查询回复规范 > 标准回复模板：`- {emoji} {技能名称}`
3. 常用业务模块表格：`| {模块名} | 通过 {skill-name} skill |`

**TOOLS.md：** 在 skill 列表末尾新增一节，含用途、脚本位置、使用方法、参数说明、重要规则

> 详见 [第 18 章 Agent 技能列表完整性配置](./18-agent-skill-list-completeness.md)

### 常见 Skill 类型

| 类型 | 示例 | 特点 |
|------|------|------|
| 需参数 + 确认 | order-archive、mbhopen-delete | 需查询参数、dry-run 预览、用户确认后再执行 |
| 需参数 + 可选确认 | ticket-restart、pboss-resend | dry-run 预览、可选 `--ids` 指定记录 |
| 需参数 + 接口调用 | file-resend | 收集需求、匹配接口、确认后执行 |

### 检查清单

- [ ] skill 名称与目录名一致
- [ ] SKILL.md 包含完整安全保密规则
- [ ] run.sh 可执行（`chmod +x`）
- [ ] 符号链接正确创建
- [ ] AGENTS.md 三处均已更新
- [ ] TOOLS.md 使用说明完整
- [ ] 默认走测试环境，生产需显式 `--env prod`
