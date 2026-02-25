# OpenClaw 运维手册

从安装部署、Agent 编排、多渠道接入（Telegram / 飞书 / 钉钉 / Web）到 Skills 开发与安全加固的完整操作手册。

> 基于 OpenClaw **2026.2.23** 编写

## 目录

1. [安装 OpenClaw](./01-install.md)
2. [核心概念](./02-concepts.md)
3. [创建 Agent Manager](./03-create-agent-manager.md)
4. [通过 Agent Manager 创建其他 Agent](./04-create-agents.md)
5. [接入 Telegram 机器人](./05-telegram.md)
6. [创建 Telegram 群组并绑定 Agent](./06-telegram-bindgroups.md)
7. [接入飞书](./07-feishu.md)
8. [为飞书绑定 Agent](./08-feishu-bindagent.md)
9. [接入钉钉自定义机器人](./09-dingtalk.md)
10. [为 Agent 创建 Skills](./10-create-skills.md)
11. [为 Skills 配置安全规则](./11-security-rules.md)
12. [Policy 权限策略配置](./12-policy.md)
13. [定时任务（Cron）配置](./13-cron.md)
14. [安全加固最佳实践](./14-security-best-practices.md)
15. [通过 HTTP API 和 Web 聊天界面访问 Agent](./15-http-web-chat.md)
16. [实战案例：ToDesk 远程连接信息复制 Skill](./16-case-todesk-skill.md)
17. [会话管理与性能优化](./17-session-management.md)
18. [Agent 技能列表完整性配置](./18-agent-skill-list-completeness.md)
19. [群聊多人并发操作的确认安全](./19-group-confirmation-safety.md)

---

## 归档规范

**本手册在归档、发布或对外分享前，必须遵守以下规范，不得泄露私有配置信息。**

### 禁止包含的内容

| 类型 | 示例 | 处理方式 |
|------|------|----------|
| 密码、密钥、Token | API Key、Bot Token、App Secret、数据库密码 | 使用占位符：`<你的 API Key>`、`<token>` |
| 数据库连接信息 | 真实 IP、端口、Service Name、用户名、密码 | 使用示例值：`10.xxx.xxx.xxx`、`myuser`、`mypassword` |
| 用户 ID、群组 ID | 飞书 open_id、Telegram User ID、群组 ID | 使用占位符：`ou_xxxxxxxx`、`123456789` |
| 本地路径 | `/Users/用户名/`、项目绝对路径 | 使用 `~`、`<username>` 或通用路径如 `~/openClawSkills/` |
| 代理端口 | 个人代理软件的实际端口 | 使用通用示例端口：`7890`、`7891` |
| Webhook URL | 钉钉、飞书等完整回调地址 | 使用模板：`https://oapi.dingtalk.com/robot/send?access_token=<token>` |

### 归档前检查清单

- [ ] 全文搜索 `password`、`PASSWORD`、`apiKey`、`token`、`secret`，确认均为占位符
- [ ] 全文搜索真实 IP（如 `10.238.`、`192.168.`），确认已替换为示例
- [ ] 全文搜索 `/Users/`，确认无真实用户名
- [ ] 路径使用 `~` 或通用路径，不使用 `Documents/cursorProj` 等个人目录
- [ ] 代理端口使用通用示例（如 7890），不使用个人代理端口

### 通用示例值

- **路径**：`~/openClawSkills/`、`~/.openclaw/workspace/<agent>/`
- **代理**：`http://127.0.0.1:7890`、`socks5://127.0.0.1:7891`
- **数据库**：`10.xxx.xxx.xxx`、`myuser`、`mypassword`、`myservice`

---

## 版本记录

### v2.7.0（2026-02-25）

**飞书路由规则定稿** — 按现网 `openclaw.json` 修正 07/08 章中飞书绑定与账号匹配说明，消除 `accountId` 放置位置和 `groups` 路由职责的歧义。

- **07 章修正**：`channels.feishu` 示例对齐现网（`dmPolicy: "disabled"`、`allowFrom: []`、`groupAllowFrom` 多群组示例）；`groups` 示例仅保留群行为控制（如 `requireMention`），移除误导性的群内路由写法；新增提示明确 `accountId` 不在 `groups` 中配置
- **08 章修正**：`bindings` 示例统一补齐 `match.accountId: "main"`（与 `match.peer` 同级）；飞书兜底/混合绑定示例同步补齐 `accountId`；“完整示例”改为 `groups` 仅控行为、`bindings` 负责路由
- **规则澄清**：统一强调 `bindings.match.accountId` 省略时仅匹配 `default`；多账号（如 `accounts.main`）必须显式写对应 `accountId`，否则可能路由 miss 并回落默认 Agent
- **安全提醒补充**：文档内新增飞书密钥/令牌风险提示，建议生产环境使用占位符与密钥管理，并对已暴露凭证及时轮换

### v2.6.0（2026-02-14）

**飞书渠道访问控制修正与安全加固** — 修正 `groupPolicy: "allowlist"` 配置方式，完成安全审计加固。

- **07 章重写 groupPolicy 章节**：修正 `groupAllowFrom` 的正确用法 — 填写**群组 ID**（`oc_xxx`）而非用户 ID（`ou_xxx`）；新增「访问控制两层模型」说明（第 1 层 groupAllowFrom 控制群组准入，第 2 层 groups.allowFrom 控制群内用户）；新增 `groupAllowFrom` 配置示例和群内用户级限制示例；更新飞书与 Telegram 配置差异对比表（新增 groupPolicy 推荐、群组白名单字段、群内用户限制对比）；7.2 配置示例改为 `groupPolicy: "allowlist"` + `groupAllowFrom`
- **08 章更新完整示例**：完整飞书绑定示例中加入 `groupPolicy: "allowlist"` 和 `groupAllowFrom` 配置，补充 groupAllowFrom 与 groups 保持一致的说明
- **14 章新增 14.8**：「渠道访问控制加固」— 飞书渠道推荐配置（dmPolicy + groupPolicy 双重白名单）、关键字段说明表（allowFrom vs groupAllowFrom vs groups.allowFrom 的区别）、网关控制台 `allowInsecureAuth: false`、文件权限收紧（policy.yaml / ip_whitelist.conf → 600）、TOOLS.md 脱敏要求；Agent 上线 Checklist 新增渠道白名单检查项
- **安全加固已部署**：`openclaw.json` 网关 `allowInsecureAuth` 已禁用、飞书 `groupPolicy` 已改为 `"allowlist"` 并通过 `groupAllowFrom` 限制为指定群组、`policy.yaml` / `ip_whitelist.conf` 文件权限已收紧为 600、`broadband-ops-agent/TOOLS.md` 已脱敏（移除数据库表名、存储过程名、脚本路径等内部细节）

### v2.5.0（2026-02-13）

**群聊多人并发操作的确认安全** — 分析群内多人同时操作不同 Skill 时确认消息被误关联的风险，设计并实施特定确认短语机制。

- **新增第 19 章**：「群聊多人并发操作的确认安全」— 问题背景、根因分析（会话按群共享不区分用户、确认机制为纯提示词约束无编程校验、串行处理的危险时序图）、风险等级评估、解决方案（特定确认短语机制 — 改动前后对比、各 Skill 确认短语、有效性分析、涉及的配置文件、局限性）、操作规范建议（避免并发/严格回复/高危走私聊/关注 ACK）、长期改进方向、适用范围
- **08 章更新**：「8.6」精简为摘要，指向第 19 章
- **5 个 Skill 确认机制改进**：将通用的"请确认"改为要求回复包含操作关键词和关键参数的特定确认短语，LLM 被指示拒绝不含关键词的通用确认
  - mbhopen-delete：确认短语改为"确认删除 SN xxx"
  - pboss-resend：确认短语改为"确认回单 记录ID"
  - ticket-restart：确认短语改为"确认重启 记录ID"
  - order-archive：确认短语改为"确认归档 工单ID"
  - file-resend：确认短语改为"确认补传 文件类型"
- **broadband-ops-agent 同步更新**：AGENTS.md「Skill 执行规范」第3条改为特定确认短语机制；TOOLS.md 中 5 个 Skill 的确认描述同步更新

### v2.4.0（2026-02-13）

**归档规范** — 新增手册归档时的私有信息保护规范。

- **README 新增**：「归档规范」— 禁止包含的内容（密码/密钥、数据库连接、用户 ID、本地路径、代理端口、Webhook URL）、归档前检查清单、通用示例值
- **配置脱敏**：01/05 章代理端口改为 7890/7891，10 章路径改为 `~/openClawSkills/`

### v2.3.0（2026-02-13）

**Cursor Plan 创建 Skill 操作说明** — 补充在 Cursor 中基于 Plan 创建 Skill 的详细步骤。

- **10 章新增**：「10.0 在 Cursor 中基于 Plan 创建 Skill（推荐）」— Plan 文件位置、4 步操作流程（打开项目、引用 Plan 描述需求、确认 Todo 执行、可选引用参考 Skill）、示例提示词、注意事项
- **10.11 更新**：标题改为「Plan 内容参考」，并增加指向 10.0 的推荐说明

### v2.2.0（2026-02-13）

**broadband-ops-agent 新建 Skill 标准流程** — 将标准 Plan 整合至手册。

- **10 章新增**：「10.11 broadband-ops-agent 新建 Skill 标准流程」— 路径约定、标准目录结构、各文件设计要点、符号链接部署、AGENTS.md/TOOLS.md 更新模板、常见 Skill 类型、检查清单
- **18 章更新**：新增技能维护流程补充「创建符号链接」「常用业务模块表格」步骤，并引用 10.11 完整流程

### v2.1.0（2026-02-13）

**第 16 章同步更新** — 与 todesk-copy-info skill v1.2.0 对齐。

- **16 章全面更新**：skill 从 v1.1.0 升级到 v1.2.0，新增锁屏自动回退功能
- **16.2 技术方案**：新增锁屏回退流程图（检测锁屏 → 读取 config.ini → 验证缓存 → 返回结果）
- **16.3 文件结构**：补充运行时缓存目录和临时文件说明
- **16.4 skill.json**：版本号更新为 1.2.0，description 新增"支持锁屏时从缓存回退"，copy 命令使用实际绝对路径
- **16.5 脚本设计**：主流程图重绘（含锁屏分支），新增「锁屏检测」（ioreg + CGSSessionScreenIsLocked）、「缓存机制」（写入时机、缓存内容、updatepasstime 验证逻辑）章节
- **16.6 SKILL.md**：新增 YAML frontmatter 说明、锁屏自动回退章节、安全规则中补充禁止透露的配置信息
- **16.7 常见问题**：新增锁屏相关问题条目
- **16.8 经验总结**：挑战表新增「屏幕锁定时 UI 自动化失效」，设计原则新增「缓存策略」
- **配置同步**：17.7 contextWindow 当前部署值从 64K 修正为 128K（与实际 openclaw.json 一致），03 章补充 subagents.maxConcurrent，08 章完整示例补充 dmPolicy/allowFrom

### v2.0.0（2026-02-13）

**渠道访问控制加固与并发调优** — 内容分散更新至第 5、7、17 章。

- **5 章更新**：Telegram `dmPolicy` 补充完整四选项（pairing / allowlist / open / disabled），示例改为 `"disabled"`，新增安全建议
- **7 章更新**：飞书 `dmPolicy` 补充 `"allowlist"` + `allowFrom` 配置方式，新增 `allowFrom` 获取方法、`groupPolicy: "allowlist"` 注意事项（会检查发送者，不适合多人群聊）、飞书与 Telegram 安全差异对比
- **17 章新增**：「并发数调优」（17.9）— `maxConcurrent` / `subagents` 推荐公式和参考值；「模型参数限制」（17.10）— `temperature` 不可在 openclaw.json 中配置、contextWindow 与 Skill 数量的关系分析

### v1.9.0（2026-02-13）

**Agent 技能列表完整性配置** — 解决 BroadbandOpsBot 技能列表不全问题。

- **问题**：skills/ 下存在 order-archive 等技能，但未在 AGENTS.md 和 TOOLS.md 中注册；Agent 回复技能列表时依赖上下文记忆，易遗漏
- **方案**：在 AGENTS.md 增加「技能列表回复规范」— 必须以 TOOLS.md 的「Skills 工具」章节为准，逐项列出，不得遗漏；若 TOOLS.md 与 skills/ 不一致，以 TOOLS.md 为准
- **order-archive 补充**：在 AGENTS.md 主要职责和标准回复模板中增加「工单正常归档」；在 TOOLS.md 中新增 order-archive 完整说明（用途、参数、执行步骤、重要规则）
- **broadband-ops/SKILL.md**：更新「本系统目前支持的功能」为：工单重启、PBOSS回单、工单归档、文件补传、主机巡检、数据库巡检
- **新增第 18 章**：「Agent 技能列表完整性配置」— 问题背景、解决方案、实施步骤、新增技能时的维护流程、与 14 章 Checklist 的衔接

### v1.8.0（2026-02-13）

**ToDesk Skill 增强** — 跨桌面支持、响应速度优化、路径修正。

- **跨桌面激活**：`activate_todesk()` 重写，优先使用 osascript `activate` + `System Events frontmost` 实现跨 macOS Spaces 切换，增加前台验证和自动重试机制，解决 ToDesk 与 Terminal 不在同一桌面时获取失败的问题
- **响应速度优化**：脚本新增结果文件（`/tmp/todesk_result.txt`）+ 完成标志（`/tmp/todesk_done.flag`）机制；SKILL.md 将原 4 步操作合并为 1 条命令（启动 + 轮询 + 读取），用每秒轮询替代固定 `sleep 8`，从复制到回复用户的延迟从 ~25 秒缩短至 ~8-10 秒
- **SKILL.md 重写**：明确禁止在 IDE 终端执行，分步指引改为合并命令方式，新增常见错误排查
- **路径修正**：skill.json 和 SKILL.md 中的脚本路径从 `~/.openclaw/skills/` 修正为 `~/.openclaw/workspace/agent-manager/skills/`
- **自动退出 Terminal**：脚本成功后自动 quit Terminal.app；`do script` 命令末尾加 `; exit` 让 shell 退出，后台退出进程通过 `</dev/null >/dev/null 2>&1` 脱离 PTY，避免 quit 时弹出"终止进程"确认框
- **16 章更新**：同步更新技术方案流程图、脚本设计解析（新增跨桌面激活、自动退出 Terminal 小节）、SKILL.md 设计说明、挑战应对表、常见问题表

### v1.7.0（2026-02-12）

**contextWindow 调优** — 缩小上下文窗口提升响应速度。

- **全局 contextWindow 调整**：volcengine/ark-code-latest 从 256K 降至 64K，LLM 推理时间大幅缩短
- **消除配置冲突**：删除 agent 级 `models.json`，统一由全局 `openclaw.json` 管理，避免 gateway 重启时两层配置互相覆盖
- **17 章新增**：「contextWindow 调优」（17.7）— 记录配置层级关系、推荐值选择、冲突规避方案
- **影响范围**：broadband-ops、ops 两个 Agent（均使用 volcengine 模型）

### v1.6.0（2026-02-12）

**文档归档与安全加固** — 将前期所有运维操作总结归档至手册。

- **新增第 17 章**：「会话管理与性能优化」— 覆盖会话生命周期、compaction 机制及其已知问题、会话满载诊断与重置、session-cleanup Skill 完整文档、飞书 blockStreaming 优化、响应慢排查流程、模型选择建议
- **14 章更新**：新增「状态查询信息泄露防护」（14.7）— 解决 Agent 在回复"当前状态"时暴露主机名、模型名、会话 ID 等敏感信息的问题，含回复规范模板和泄露测试用例
- **14 章更新**：Agent 上线 Checklist 新增「AGENTS.md 含状态查询回复规范」检查项
- **安全规则部署**：已为 broadband-ops、agent-manager、ops 三个 Agent 的 AGENTS.md 添加「状态查询回复规范」

### v1.5.0（2026-02-12）

**session-cleanup v2.0 — 满载会话自动重置** — Skill 升级为全量会话维护。

- **重置原因**：OpenClaw 2026.2.9 的 `safeguard` compaction 从未成功触发（所有 agent 的 `compactionCount` 均为 0），渠道持久会话在长期使用后会撑满 256K 上下文窗口，导致响应极慢（实测 avg 24s / max 201s）甚至超时，必须通过外部重置来保障响应效率
- **新增满载检测**：`overloaded` 命令检测上下文使用率 ≥80% 的会话（包括受保护的渠道会话）
- **新增满载重置**：`reset-overloaded` / `reset-overloaded-dry` 命令重置满载会话，解决响应慢问题
- **新增自动维护**：`auto` 命令一次执行"清理过期 + 重置满载"，crontab 已切换为 `--auto` 模式
- **status 增强**：状态概览现在包含满载告警（🔴 标记 ≥80% 的会话）
- **维护范围变更**：受保护会话（telegram/feishu/main）不再永不清理，上下文满载时自动重置

### v1.4.0（2026-02-12）

**全局满载会话重置** — 清理所有 100% 上下文窗口的会话。

- **agent-manager Telegram 群**：重置 256K/256K 满载会话（670KB），`compactionCount: 0`（safeguard 未生效）
- **main 主控台**：重置 256K/256K 满载会话（829KB），`inputTokens: 480,667`
- **safeguard 排查**：确认当前版本 safeguard compaction 从未成功触发（所有 agent 的 compactionCount 均为 0）
- **重置后状态**：全部会话使用率降至 42% 以下，无红色告警

### v1.3.0（2026-02-12）

**飞书响应优化** — 解决飞书群 Agent 响应慢的问题。

- **飞书会话重置**：清空已撑满 256K 上下文窗口的历史会话（631KB / 302 轮），恢复响应速度
- **关闭 blockStreaming**：飞书渠道从阻塞模式改为流式推送，用户可逐步看到回复而非等待全部生成
- **compaction 说明**：确认 `safeguard` 是当前版本唯一可用的 compaction 模式，`auto` 等值不被支持

### v1.2.0（2026-02-12）

**会话回收 Skill 化** — 将独立脚本重构为标准 OpenClaw Skill。

- **15 章更新**："会话回收"小节改为 Skill 使用方式，推荐通过 Agent Manager 自然语言调用
- **新增 Skill**：`session-cleanup`（Agent Manager workspace skills），包含 6 个命令（status / dry-run / cleanup / cleanup-ttl / cleanup-agent / logs）
- **安全加固**：SKILL.md 添加完整安全规则，脚本文件添加 CONFIDENTIAL 标记

### v1.1.0（2026-02-12）

**会话回收机制** — 修复 Gateway 会话无限增长问题。

- **15 章修正**：移除不准确的"4 小时自动过期"描述，改为真实的会话生命周期说明
- **15 章新增**："会话回收"小节 — 记录会话清理机制的部署方式
- **部署清理**：4 小时 TTL 自动回收临时会话，保护渠道持久会话（telegram/feishu）

### v1.0.0（2026-02-12）

**初始版本** — 手册创建，涵盖 OpenClaw 2026.2.9 全流程。

新增章节：

- **01 安装 OpenClaw** — Homebrew 安装、gateway 启动、基础配置
- **02 核心概念** — Agent、Skill、Channel、Gateway、Session 等概念说明
- **03 创建 Agent Manager** — defaults 配置、模型选择、compaction 模式
- **04 通过 Agent Manager 创建其他 Agent** — 多 Agent 编排、workspace 隔离
- **05 接入 Telegram** — BotFather 创建机器人、配置 botToken
- **06 Telegram 群组绑定** — 创建群组并将 Agent 绑定到指定群
- **07 接入飞书** — 飞书开放平台应用创建、事件订阅配置
- **08 飞书绑定 Agent** — 飞书群组与 Agent 的绑定关系配置
- **09 接入钉钉** — 钉钉自定义机器人接入流程
- **10 创建 Skills** — Skill 目录结构、SKILL.md 编写规范、skill.json 配置
- **11 Skills 安全规则** — 文件访问控制、exec 权限、allowlist 机制
- **12 Policy 权限策略** — policy.yaml 编写、工具/文件/技能三层权限模型
- **13 定时任务（Cron）** — cron 表达式、jobs.json 配置、任务调度
- **14 安全加固最佳实践** — 网络隔离、token 管理、最小权限原则
- **15 HTTP API 和 Web 聊天** — chat-server.py 部署、OpenAI 兼容 API、Web UI
- **16 实战案例：ToDesk Skill** — 端到端 Skill 开发案例，含脚本、安全规则、测试
