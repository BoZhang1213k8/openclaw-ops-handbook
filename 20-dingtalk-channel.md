# 20. 接入钉钉应用机器人（对话通道）

## 本章定位（渠道级对话接入）

本章说明的是 OpenClaw `channels.dingtalk` 的接入方式：  
钉钉作为**对话渠道**接入 Gateway，用户可直接在钉钉私聊/群聊中与 Agent 交互。

> 与 [第 9 章](./09-dingtalk.md) 的区别：  
> 第 9 章是 Skill 脚本通过 Webhook 主动推送（单向通知）；  
> 本章是 OpenClaw 渠道插件接入（双向对话）。

> 说明：本章为一线落地经验总结，面向团队分享。  
> 文中的“必须/推荐”均基于已验证环境，跨租户请按实际权限模型与平台文档复核。

## 20.1 配置总结（可复用样例）

核心配置如下：

| 配置项 | 示例值 | 说明 |
|------|------|------|
| `plugins.entries.dingtalk.enabled` | `true` | 启用 dingtalk 插件入口 |
| `plugins.installs.dingtalk.spec` | `@soimy/dingtalk` | 插件来源（npm） |
| `plugins.installs.dingtalk.version` | `3.2.0` | 示例中使用的插件版本 |
| `channels.dingtalk.enabled` | `true` | 启用钉钉渠道 |
| `channels.dingtalk.clientId` | 已配置 | 钉钉应用 clientId |
| `channels.dingtalk.clientSecret` | 已配置 | 钉钉应用 clientSecret |
| `channels.dingtalk.robotCode` | 已配置 | 机器人编码（通常与 appKey/clientId 对齐） |
| `channels.dingtalk.corpId` | 已配置 | 企业 corpId |
| `channels.dingtalk.agentId` | 已配置 | 应用 agentId |
| `channels.dingtalk.dmPolicy` | `"open"` | 允许私聊消息 |
| `channels.dingtalk.groupPolicy` | `"open"` | 允许群聊消息 |
| `channels.dingtalk.messageType` | `"markdown"` | 回复消息格式 |
| `channels.dingtalk.debug` | `false` | 关闭调试日志 |

说明：
- 未配置 `bindings` 的钉钉专属路由规则时，消息会回落到默认 Agent（由 `agents.list[].default: true` 决定）。
- 需要“不同群路由到不同 Agent”时，补充 `bindings` 规则。

## 20.2 安装钉钉插件（@soimy/dingtalk）

如果是首次接入钉钉渠道，先安装插件再写 `channels.dingtalk` 配置。

### 步骤 1：安装插件

```bash
openclaw plugin add @soimy/dingtalk
```

安装成功后，可在 `openclaw.json` 看到类似结构：

```jsonc
"plugins": {
  "entries": {
    "dingtalk": { "enabled": true }
  },
  "installs": {
    "dingtalk": {
      "source": "npm",
      "spec": "@soimy/dingtalk",
      "version": "3.2.0"
    }
  }
}
```

### 步骤 2：启用插件入口

确认：

```jsonc
"plugins": {
  "entries": {
    "dingtalk": {
      "enabled": true
    }
  }
}
```

### 步骤 3：重启 Gateway 并验证加载

- 重启 OpenClaw Gateway
- 检查启动日志，确认没有 dingtalk 插件加载报错

### 升级/重装建议

- 升级插件：重新执行 `openclaw plugin add @soimy/dingtalk`
- 异常重装：先删除对应安装目录，再执行安装命令
- 升级后务必回归验证：私聊、群聊、消息格式三项

## 20.3 钉钉开发者平台配置（创建应用/机器人/权限）

本节用于把钉钉侧前置动作一次做全，避免“字段都填了但不回消息”。

### 20.3.1 创建企业内部应用

1. 打开钉钉开发者平台，进入企业组织
2. 进入“应用开发”创建应用（企业内部应用）
3. 填写应用名称、描述、可见范围并保存

创建完成后，记录以下信息（后续写入 `channels.dingtalk`）：
- 应用 `Client ID`（AppKey）→ `clientId`
- 应用 `Client Secret`（AppSecret）→ `clientSecret`
- 应用 `AgentId` → `agentId`
- 企业 `CorpId` → `corpId`

### 20.3.2 添加机器人能力

1. 进入应用配置页，打开“机器人”能力
2. 开启机器人，并确认可在企业内接收私聊/群聊消息
3. 记录机器人编码（`robotCode`）
4. 在“事件与回调/事件订阅”中将事件推送模式设置为 **Stream**

字段对应关系：
- 机器人编码 → `channels.dingtalk.robotCode`

> 关键要求：事件推送必须是 **Stream 模式**，否则可能出现消息无法稳定进入 OpenClaw 的情况。

### 20.3.3 添加并开通权限

按本手册的已验证经验，手动开通以下两个权限即可：

- `Card.Instance.Write`
- `Card.Streaming.Write`

其他权限通常无需手动单独开通也可使用（以租户默认授权与应用发布后的实际生效为准）。

操作要点：
- 权限添加后提交审核/发布
- 权限生效前，机器人可能“能收到事件但无法回复”
- 若仅使用 `messageType: "markdown"`，这两个 Card 权限可作为统一权限基线保留

### 20.3.4 发布版本并校验可见范围

1. 在开发者平台发布应用新版本（测试/正式按企业流程）
2. 检查应用可见范围是否包含目标用户和目标群
3. 在钉钉客户端把机器人加入目标群

若未发布或可见范围未覆盖，OpenClaw 侧配置正确也无法稳定收发消息。

## 20.4 OpenClaw 侧配置步骤（渠道模式）

1. 安装并启用 dingtalk 插件（`@soimy/dingtalk`）
2. 在钉钉开发者平台确认事件推送为 **Stream 模式**
3. 在 `openclaw.json` 的 `channels.dingtalk` 写入字段
4. 重启 OpenClaw Gateway
5. 先做私聊验证，再做群聊验证

## 20.5 配置模板

```jsonc
{
  "plugins": {
    "entries": {
      "dingtalk": {
        "enabled": true
      }
    },
    "installs": {
      "dingtalk": {
        "source": "npm",
        "spec": "@soimy/dingtalk",
        "version": "3.2.0"
      }
    }
  },
  "channels": {
    "dingtalk": {
      "enabled": true,
      "clientId": "<DINGTALK_CLIENT_ID>",
      "clientSecret": "<DINGTALK_CLIENT_SECRET>",
      "robotCode": "<DINGTALK_ROBOT_CODE>",
      "corpId": "<DINGTALK_CORP_ID>",
      "agentId": "<DINGTALK_AGENT_ID>",
      "dmPolicy": "open",
      "groupPolicy": "open",
      "messageType": "markdown",
      "debug": false
    }
  }
}
```

> 安全建议：生产环境优先使用环境变量注入，不要在文档或仓库中保留真实密钥。

## 20.6 路由与 Agent 绑定建议

### 方案 A：先用默认 Agent 承接（通用起步方案）

- 不额外配置钉钉 `bindings`
- 由默认 Agent 统一处理钉钉来消息
- 适合初始阶段快速验证连通性

### 方案 B：按群/场景分流（推荐生产）

- 为钉钉补充 `bindings`，按群组或会话来源匹配到指定 Agent
- 管理类问题路由到 `agent-manager`
- 业务执行问题路由到 `ops` / `broadband-ops`

落地原则：
- 先收敛一条主路由，再逐步拆分
- 单次只引入一个新规则，便于回归与排障

## 20.7 验证清单

- [ ] 钉钉插件已安装且 `plugins.entries.dingtalk.enabled = true`
- [ ] 钉钉事件推送模式为 **Stream**
- [ ] `channels.dingtalk` 必填字段完整且无拼写错误
- [ ] 钉钉后台已手动开通 `Card.Instance.Write` 与 `Card.Streaming.Write`
- [ ] Gateway 启动日志无 dingtalk 鉴权/初始化报错
- [ ] 私聊与群聊都能收到 Agent 回复（与 `dmPolicy` / `groupPolicy` 一致）
- [ ] 回复格式符合 `messageType: "markdown"` 预期

## 20.8 常见问题排查

### 启动失败或鉴权失败

- 优先检查 `clientId/clientSecret/corpId/agentId/robotCode` 是否对应同一个应用
- 检查应用权限与机器人可见范围是否已在钉钉侧生效
- 权限侧按现网基线先确认 `Card.Instance.Write`、`Card.Streaming.Write` 已开通

### 能收到消息但不回复

- 检查是否被策略拦截（`dmPolicy` / `groupPolicy`）
- 检查是否没有命中预期路由（未配置 `bindings` 时会走默认 Agent）
- 回到钉钉后台确认事件推送仍为 **Stream 模式**

### 回复格式异常

- `messageType` 建议优先使用 `"markdown"`
- 避免复杂富文本语法，先用纯文本/基础 Markdown 验证链路

## 下一步

- 若你需要“群级别精细路由”，继续参考 [第 8 章](./08-feishu-bindagent.md) 的绑定思路，把同样的方法迁移到钉钉 `bindings` 规则中。
- 若你要做“任务执行后主动通知钉钉群”，参考 [第 9 章](./09-dingtalk.md) 的 Webhook 推送模式。
