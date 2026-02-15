# 16. 实战案例：ToDesk 远程连接信息复制 Skill

本章通过一个完整的实际案例，展示如何从零开发一个 macOS 自动化 Skill——自动从 ToDesk 主界面复制远程连接信息（设备代码 + 临时密码 + 远程链接）到剪贴板，并支持锁屏时从缓存自动回退。

## 16.1 需求背景

日常运维中，经常需要将本机的 ToDesk 远程连接信息发送给同事，让他们远程协助。手动操作需要：

1. 打开 ToDesk
2. 找到设备代码旁的复制按钮
3. 点击复制
4. 粘贴到聊天窗口

通过 Skill，只需在群组中对 Agent 说一句"复制 ToDesk 连接信息"，Agent 自动完成以上所有步骤并返回结果。

## 16.2 技术方案

### 正常流程（屏幕未锁定）

```
用户消息: "发一下 ToDesk 连接信息"
       ↓
Agent Manager 识别意图 → 调用 todesk-copy-info skill
       ↓
osascript 打开本地终端（Terminal.app）
       ↓
bash 脚本执行:
  1. 检测屏幕是否锁定 → 未锁定，走 UI 自动化流程
  2. 检查 ToDesk 是否运行，未运行则自动启动
  3. 激活 ToDesk 窗口（支持跨桌面/Spaces 切换）
  4. 使用 Peekaboo 获取 ToDesk 窗口位置
  5. 计算复制按钮的屏幕坐标
  6. 使用 Peekaboo 点击复制按钮
  7. 读取剪贴板内容
  8. 缓存连接信息（供锁屏时使用）
  9. 写入结果文件 + 完成标志
       ↓
Agent 轮询完成标志 → 读取结果文件 → 回复用户
```

### 锁屏回退流程

```
脚本检测到屏幕已锁定
       ↓
UI 自动化无法工作，进入回退模式
       ↓
读取 ToDesk config.ini → 获取设备代码 + 密码更新时间
       ↓
检查本地缓存 (~/.cache/todesk-copy-info/)
       ↓
  ┌─ 缓存有效（updatepasstime 匹配）→ 返回完整连接信息（设备代码 + 密码 + 链接）
  └─ 缓存无效或不存在 → 仅返回设备代码，提示需解锁屏幕获取密码
```

**核心依赖：**

| 工具 | 作用 |
|------|------|
| [Peekaboo](https://github.com/nicklama/peekaboo) | macOS 窗口管理和 UI 自动化工具，支持窗口定位、坐标点击 |
| osascript | macOS AppleScript 引擎，用于窗口切换和终端调用 |
| pbcopy / pbpaste | macOS 剪贴板工具 |
| python3 | 解析窗口 JSON 数据、计算坐标、读取配置文件 |
| ioreg | macOS 系统工具，用于检测屏幕锁定状态 |

## 16.3 文件结构

```
todesk-copy-info/
├── skill.json                     # 技能元数据和命令定义
├── SKILL.md                       # Agent 使用指南 + 安全规则
└── scripts/
    └── copy_todesk_info.sh        # 主脚本（Bash + 嵌入 Python）
```

运行时产生的缓存和临时文件：

```
~/.cache/todesk-copy-info/         # 持久缓存目录
├── last_result.txt                # 上次成功获取的完整连接信息
└── last_meta.txt                  # 上次的 updatepasstime（用于验证缓存是否过期）

/tmp/todesk_result.txt             # 本次执行结果（Agent 读取）
/tmp/todesk_done.flag              # 完成标志（OK 或 FAIL）
/tmp/todesk_output.log             # 终端完整输出日志（调试用）
/tmp/todesk_peekaboo/              # Peekaboo 窗口信息临时目录
```

这个 Skill 没有 `.env` 文件——它不需要外部凭证，所有操作都在本地完成。

## 16.4 skill.json 解析

```json
{
  "name": "todesk-copy-info",
  "version": "1.2.0",
  "description": "ToDesk 连接信息复制 - 自动从 ToDesk 主界面复制设备代码和临时密码到剪贴板，支持锁屏时从缓存回退",
  "author": "Alan Zhang",
  "tags": ["todesk", "remote-desktop", "automation", "macos"],
  "requirements": {
    "platforms": ["darwin"],
    "dependencies": ["peekaboo", "python3"]
  },
  "commands": [
    {
      "name": "copy",
      "description": "复制 ToDesk 连接信息到剪贴板（在新终端窗口中执行）",
      "executable": "osascript",
      "arguments": [
        "-e", "tell application \"Terminal\" to activate",
        "-e", "tell application \"Terminal\" to do script \"bash ~/.openclaw/workspace/agent-manager/skills/todesk-copy-info/scripts/copy_todesk_info.sh 2>&1 | tee /tmp/todesk_copy_output.log; exit\""
      ]
    },
    {
      "name": "copy-direct",
      "description": "直接执行脚本复制 ToDesk 连接信息到剪贴板",
      "executable": "bash",
      "arguments": ["scripts/copy_todesk_info.sh"]
    },
    {
      "name": "show-output",
      "description": "显示上次执行结果",
      "executable": "cat",
      "arguments": ["/tmp/todesk_copy_output.log"]
    }
  ]
}
```

### 关键设计点

**1. 为什么需要两个执行命令（copy vs copy-direct）？**

| 命令 | 执行方式 | 使用场景 |
|------|----------|----------|
| `copy` | 通过 osascript 打开 Terminal.app 执行 | 通过 Telegram/飞书远程触发（推荐） |
| `copy-direct` | 直接在当前 shell 执行 | 本地调试 |

原因：macOS 的**屏幕录制**和**辅助功能**权限是按应用绑定的。Peekaboo 在读取窗口信息时需要调用方拥有屏幕录制权限，而 OpenClaw 网关进程运行在沙盒环境中，**无法获取屏幕录制权限**——即使给 Peekaboo 本身授权了，在沙盒中调用它时依然会因为调用方（网关进程）没有权限而读取不到屏幕内容。

解决方案：通过 osascript 打开一个**已被授予屏幕录制权限的 Terminal.app 窗口**来执行脚本。Terminal.app 作为调用方拥有权限，Peekaboo 就能正常工作了。

**2. `do script` 末尾的 `; exit`**

```
do script "bash script.sh 2>&1 | tee /tmp/log; exit"
```

脚本执行完后 `exit` 让 shell 退出，配合脚本内的 `close_terminal()` 函数自动 quit Terminal.app，保持桌面整洁。

**3. 异步执行的结果获取机制**

`copy` 命令在新终端窗口中异步执行，Agent 无法直接获取输出。脚本通过以下机制传递结果：

- **结果文件 + 完成标志**（主要机制）：脚本完成后写入 `/tmp/todesk_result.txt`（纯连接信息）和 `/tmp/todesk_done.flag`（标志文件，内容为 `OK` 或 `FAIL`）。Agent 通过轮询标志文件判断是否完成（每秒检查一次，最多 20 秒），完成后直接读取结果文件。这比固定 `sleep` 更高效——脚本提前完成就提前返回。
- **日志文件**（调试用）：`tee` 将完整终端输出写入日志文件，脚本失败时 Agent 可读取日志排查原因。

> **注意：** skill.json 中保留的 `show-output` 命令是早期的结果读取方式（通过 `cat` 读日志文件），现已由结果文件 + 轮询机制替代，仅作为手动调试时的回退手段。

**4. 日志文件路径说明**

skill.json 的 `copy` 命令和 SKILL.md 中 Agent 的执行命令使用了不同的日志文件名：

| 来源 | 日志路径 | 使用场景 |
|------|----------|----------|
| skill.json `copy` 命令 | `/tmp/todesk_copy_output.log` | OpenClaw 框架直接调用命令时写入 |
| SKILL.md Agent 指令 | `/tmp/todesk_output.log` | Agent 读取 SKILL.md 后自行构造命令时写入 |

两者不冲突——实际由 Agent 执行时，以 SKILL.md 中的路径为准。`show-output` 命令读取的是 skill.json 的日志路径。

**5. 平台限制**

`requirements.platforms` 设为 `["darwin"]`——此 Skill 仅适用于 macOS，因为依赖 osascript、Peekaboo、pbcopy 等 macOS 专有工具。

## 16.5 脚本设计解析

### 主流程

```
main()
  ├── is_screen_locked()     → 检测屏幕是否锁定
  │     ├── 锁定 → lockscreen_fallback()
  │     │            ├── read_config_info()  → 从 config.ini 读设备代码 + updatepasstime
  │     │            ├── read_cache()        → 验证并读取缓存
  │     │            │     ├── 缓存有效 → 返回完整连接信息
  │     │            │     └── 缓存无效 → 返回部分信息（仅设备代码）
  │     │            └── 输出结果 + 写入标志 + close_terminal()
  │     │
  │     └── 未锁定 → 正常 UI 自动化流程 ↓
  │
  ├── preflight()                → 检查 Peekaboo 是否安装、ToDesk 是否运行
  ├── activate_todesk()          → 激活 ToDesk 窗口（支持跨桌面 Spaces 切换）
  ├── calc_copy_button_coords()  → 获取窗口坐标 → 计算复制按钮位置
  ├── click_and_read()           → 点击按钮 → 读取剪贴板 → 重试机制
  ├── save_cache()               → 缓存成功结果（供锁屏时使用）
  └── close_terminal()           → 脱离 PTY 后延迟退出 Terminal.app
```

### 锁屏检测

通过 `ioreg` 查询 macOS 系统级别的屏幕锁定状态：

```python
# 使用 plistlib 解析 ioreg 输出
r = subprocess.run(['ioreg', '-n', 'Root', '-d1', '-a'], capture_output=True)
d = plistlib.loads(r.stdout)
users = d.get('IOConsoleUsers', [])
for u in users:
    if u.get('CGSSessionScreenIsLocked', False):
        print('locked')
```

`CGSSessionScreenIsLocked` 是 macOS CoreGraphics 的内部标志，在屏幕锁定（合盖、锁屏快捷键、自动锁定）时为 `True`。

### 缓存机制

缓存用于在屏幕锁定时提供连接信息（因为 UI 自动化无法在锁屏下工作）。

**写入时机：** 每次 UI 自动化成功获取连接信息后，同时保存到缓存目录。

**缓存内容：**

| 文件 | 内容 |
|------|------|
| `~/.cache/todesk-copy-info/last_result.txt` | 完整连接信息（设备代码 + 密码 + 链接） |
| `~/.cache/todesk-copy-info/last_meta.txt` | 缓存时的 `updatepasstime` 值 |

**验证逻辑：** ToDesk 每次更新临时密码时，`config.ini` 中的 `updatepasstime` 字段会变化。读取缓存时，比较当前的 `updatepasstime` 与缓存时记录的值：

```
当前 updatepasstime == 缓存的 updatepasstime → 密码未变，缓存有效
当前 updatepasstime != 缓存的 updatepasstime → 密码已更新，缓存失效
```

### 自动启动处理

```bash
# 如果 ToDesk 未运行，自动启动
if ! pgrep -f "ToDesk" &>/dev/null; then
    open -a "ToDesk"
    sleep 5     # 冷启动需要更长时间等待窗口完全加载
fi
```

### 跨桌面（Spaces）激活

当 ToDesk 和 Terminal.app 不在同一个 macOS 桌面时，简单的 `peekaboo app switch` 无法正确切换，Peekaboo 也获取不到窗口信息。解决方案：

1. **优先使用 osascript 激活** — AppleScript 的 `activate` 命令会让 macOS 自动切换到目标应用所在的 Space，这是跨桌面切换最可靠的方式
2. **双重保障** — 先 `activate` 应用，再通过 `System Events` 将进程设为 `frontmost`
3. **充足的等待时间** — macOS Space 切换动画约 0.7 秒，等待 2 秒确保切换完成
4. **前台验证 + 重试** — 激活后检查当前前台应用是否为 ToDesk，不是则再次尝试

```bash
# 核心逻辑：osascript 激活 + System Events frontmost + 验证重试
osascript -e '
    tell application "ToDesk" to activate
    delay 1
    tell application "System Events"
        tell process "ToDesk"
            set frontmost to true
        end tell
    end tell
'
sleep 2  # 等待 Space 切换动画

# 验证是否成功激活，失败则重试
frontapp=$(osascript -e 'tell application "System Events" to get name of first application process whose frontmost is true')
if [[ "$frontapp" != *"ToDesk"* ]]; then
    # 再次尝试激活...
fi
```

### 窗口坐标计算（嵌入 Python）

脚本使用 Peekaboo 获取窗口边界信息（JSON），然后用 Python 计算复制按钮的绝对屏幕坐标：

```python
# 复制按钮相对于窗口的位置（通过实测校准）
copy_x = int(win_x + win_w * 0.45)    # 水平 45%
copy_y = int(win_y + win_h * 0.25)    # 垂直 25%
```

> **经验：** UI 自动化中的坐标需要实际测量校准。相对坐标（百分比）比绝对坐标更稳定，能适应不同窗口大小。

### 自动退出 Terminal.app

脚本通过 `osascript` 在 Terminal.app 中异步执行，执行完毕后 Terminal 窗口会留在屏幕上。为保持桌面整洁，脚本成功后自动退出 Terminal.app。

实现中有两个关键点：

**1. `do script` 命令末尾加 `; exit`**

```
do script "bash script.sh 2>&1 | tee /tmp/log; exit"
```

脚本执行完后 `exit` 让 shell 退出，Terminal 中不再有活跃的 shell 进程。

**2. 后台退出进程必须脱离 PTY**

```bash
(sleep 3 && osascript -e 'tell application "Terminal" to quit') </dev/null >/dev/null 2>&1 & disown
```

`</dev/null >/dev/null 2>&1` 是关键——如果不加，后台进程会继承 Terminal 的 PTY（伪终端）文件描述符。即使 shell 已退出，Terminal 检测到仍有进程持有 PTY，`quit` 时会弹出 **"Do you want to terminate running processes in this window?"** 确认对话框。完全脱离 PTY 后，`quit` 可以静默退出。

### 点击重试机制

UI 自动化不一定每次都精准命中，脚本实现了自动重试：

```
第 1 次：精确坐标点击
    ↓ 失败（剪贴板为空）
第 2 次：向左偏移 20px 重试
    ↓ 失败
第 3 次：向右偏移 15px 重试
    ↓ 仍然失败 → 提示用户手动操作
```

### 输出格式

正常模式：

```
╔══════════════════════════════════════════════╗
║       ToDesk 连接信息（已在剪贴板）          ║
╠══════════════════════════════════════════════╣
║  xxx邀请您进行远程控制                       ║
║  ToDesk设备代码:xxx                          ║
║  临时密码:xxx                                ║
║  点击链接直接进行远程控制：                    ║
║  https://...                                 ║
╚══════════════════════════════════════════════╝
```

锁屏模式（缓存有效时输出相同内容，标题改为"锁屏模式 - 来自缓存"）。

## 16.6 SKILL.md 设计

### YAML Frontmatter

SKILL.md 使用 YAML frontmatter 提供结构化元数据，便于 OpenClaw 框架匹配用户意图：

```yaml
---
name: todesk-copy-info
description: Copy ToDesk remote connection info (device code + temporary password)
  from the ToDesk main window on macOS. Use when the user asks to get ToDesk
  connection info, copy ToDesk password, check ToDesk device code, or share
  ToDesk remote access credentials.
---
```

> **设计要点：** `description` 使用英文编写，涵盖多种触发措辞（get info、copy password、check device code、share credentials），提高意图匹配率。

### Agent 使用指南

告诉 Agent 如何正确执行此 Skill：

```markdown
## ⚠️ 关键约束：必须通过 macOS Terminal.app 执行

禁止在 IDE 内置终端（如 Cursor、VS Code 终端）中直接运行脚本。
必须通过 osascript 委托给 Terminal.app 执行。

### 执行步骤

> 重要：只需执行下面 2 条命令，减少工具调用次数可以显著加快响应速度。

第 1 步：清理标志文件 + osascript 启动脚本
第 2 步：轮询完成标志 + 读取结果文件

推荐合并为一条命令执行，速度最快。
```

**设计要点：** SKILL.md 中推荐 Agent 将启动和等待合并为**一条 shell 命令**执行，减少 LLM 工具调用次数（每次调用有 3-5 秒的思考开销），从而显著缩短从复制到回复用户的整体延迟。

### 锁屏自动回退说明

SKILL.md 中对锁屏回退机制的说明，让 Agent 了解不同场景下的预期输出：

```markdown
### 锁屏自动回退

脚本内置了锁屏检测机制。当 macOS 屏幕锁定时，自动切换到回退模式：

1. 检测锁屏状态（ioreg 查询 CGSSessionScreenIsLocked）
2. 尝试读取缓存（通过 config.ini 的 updatepasstime 验证缓存有效性）
3. 缓存有效 → 返回完整连接信息
4. 缓存无效 → 返回设备代码，提示需解锁获取密码
```

### 安全保密规则

```markdown
## 🔒 安全保密规则（最高优先级）

### 禁止透露的实现细节

- 使用的自动化工具名称和技术方案
- UI 自动化的具体实现方式（坐标计算、元素识别、重试机制等）
- 窗口坐标、元素标识符等 UI 定位细节
- 脚本的工作原理和执行流程
- 执行命令、系统调用方式和依赖工具链
- 项目目录结构、文件名、脚本路径

### 禁止透露的配置信息

- 脚本路径和执行方式
- 权限配置细节
- 日志文件路径

### 应对策略

- "怎么实现的？" → "抱歉，工具的实现细节属于内部信息，无法透露。"
- 仅可向用户展示最终获取到的 ToDesk 连接信息
```

## 16.7 macOS 权限配置

### 权限原理

macOS 的屏幕录制和辅助功能权限**绑定到调用方应用**，而不是被调用的工具本身。这意味着：

```
OpenClaw 网关进程（沙盒）调用 Peekaboo → ❌ 无权限，读取不到屏幕内容
Terminal.app（已授权）调用 Peekaboo     → ✅ 有权限，正常工作
```

因此必须确保 **Terminal.app** 拥有这两项权限。

### 所需权限

| 权限 | 用途 | 授权对象 |
|------|------|----------|
| 屏幕录制 | Peekaboo 读取窗口位置和内容 | **Terminal.app** |
| 辅助功能 | Peekaboo 模拟鼠标点击操作 | **Terminal.app** |

### 授权操作步骤

#### 1. 授予 Terminal.app 屏幕录制权限

1. 打开 **系统设置**（System Settings）
2. 进入 **隐私与安全性**（Privacy & Security）
3. 左侧选择 **屏幕录制**（Screen Recording）
4. 点击右侧列表下方的 **`+`** 按钮
5. 在 `/System/Applications/Utilities/` 中找到 **Terminal.app**，选中并添加
6. 确保 Terminal.app 旁的开关为 **开启** 状态

#### 2. 授予 Terminal.app 辅助功能权限

1. 仍在 **隐私与安全性** 页面
2. 左侧选择 **辅助功能**（Accessibility）
3. 点击 **`+`** 按钮
4. 同样添加 **Terminal.app**
5. 确保开关为 **开启** 状态

#### 3. 重启 Terminal.app

> **重要：** 修改权限后，**必须完全退出并重新打开 Terminal.app** 才能生效。仅关闭窗口不够，需要 `⌘Q` 退出应用后重新启动。

#### 4. 验证权限

在 Terminal.app 中执行：

```bash
# 测试屏幕录制权限（应该能列出窗口信息）
peekaboo list windows --app "Finder" --json

# 测试辅助功能权限（应该能模拟点击）
peekaboo click --coords "100,100" --app "Finder"
```

如果返回正常结果，说明权限配置成功。如果报权限错误，检查系统设置中的开关是否已开启，并确认已重启 Terminal.app。

### 使用 iTerm2 替代 Terminal.app

如果你使用 iTerm2 作为终端，需要将上述所有步骤中的 Terminal.app 替换为 iTerm2，同时修改 `skill.json` 中的 osascript 命令：

```json
{
  "name": "copy",
  "executable": "osascript",
  "arguments": [
    "-e", "tell application \"iTerm\" to activate",
    "-e", "tell application \"iTerm\" to create window with default profile command \"bash <脚本路径>/copy_todesk_info.sh 2>&1 | tee /tmp/todesk_copy_output.log\""
  ]
}
```

### 常见问题

| 现象 | 原因 | 解决 |
|------|------|------|
| Peekaboo 返回空窗口列表 | Terminal.app 没有屏幕录制权限 | 按上述步骤授权 |
| 点击命令无响应 | Terminal.app 没有辅助功能权限 | 按上述步骤授权 |
| 授权后仍不生效 | 未重启 Terminal.app | 完全退出（⌘Q）后重新打开 |
| 权限列表中找不到 Terminal | 手动输入路径 | `/System/Applications/Utilities/Terminal.app` |
| 通过 Telegram 触发无效 | 网关进程在沙盒中无权限 | 确保使用 `copy` 命令（通过 osascript 间接执行），而非 `copy-direct` |
| quit Terminal 时弹出确认框 | 后台进程继承了 PTY 文件描述符 | 后台进程启动时加 `</dev/null >/dev/null 2>&1` 脱离 PTY |
| 锁屏时返回 `[屏幕锁定，无法获取]` | 无有效缓存 | 需要先在屏幕解锁时成功执行一次，建立缓存 |

## 16.8 开发此类 Skill 的经验总结

### 适用场景

macOS UI 自动化 Skill 适合：
- 需要操作 GUI 应用（没有 CLI 或 API）
- 操作步骤固定且可预测
- 需要频繁重复执行

### 关键挑战和应对

| 挑战 | 应对方案 |
|------|----------|
| macOS 权限限制 | 通过 osascript 调用已授权终端间接执行 |
| UI 元素定位不稳定 | 使用相对坐标（百分比），添加重试机制 |
| 异步执行无法获取输出 | 写入结果文件 + 完成标志，Agent 轮询等待 |
| Agent 多步调用延迟高 | SKILL.md 指导 Agent 合并为单条命令，减少 LLM 调用开销 |
| 应用未运行 | 脚本自动启动并等待（冷启动 5 秒） |
| 窗口在不同桌面（Spaces） | 优先用 osascript activate 跨桌面切换，加验证重试 |
| 执行后 Terminal 残留在桌面 | 脚本完成后自动 quit Terminal（`; exit` + PTY 脱离 + 延迟 quit） |
| 屏幕锁定时 UI 自动化失效 | 锁屏检测 + 缓存回退（config.ini updatepasstime 验证缓存有效性） |

### Skill 设计原则

1. **优雅降级** — 每个步骤失败都有清晰的错误信息；屏幕锁定时自动回退到缓存
2. **自动恢复** — 应用未运行则自动启动，点击未命中则自动重试
3. **日志完整** — stderr 输出完整流程日志，stdout 只输出最终结果
4. **安全封装** — 用户只看到结果，不接触实现细节
5. **缓存策略** — 成功结果持久化缓存，通过业务字段（updatepasstime）验证有效性，而非简单的时间过期
