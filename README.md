# Arc Daily Task - Claude Code Skill

自动完成 **Arc Community** (community.arc.network) 每日积分任务的 Claude Code 技能。

## 积分规则

| 操作 | 数量 | 积分 |
|------|:----:|:----:|
| 📖 阅读博客/资源 | 5 篇 | 10 分（+2/篇） |
| 📺 观看视频 | 4 个 | 16 分（+4/个） |
| 📅 每日签到 | 自动 | ~1 分 |
| **每日上限** | | **≈ 26~27 分** |

> ⚠️ 规则可能变化，以 Arc 官方为准。
> 同一内容 24h 内重复做不给分。

## 安装步骤

### 1. 下载技能

```bash
# 克隆到 Claude Code 的 skills 目录
git clone https://github.com/tz8899/arc-daily-task.git ~/.claude/skills/arc-daily-task
```

或者手动复制到 `~/.claude/skills/arc-daily-task/` 目录下。

### 2. 启动调试 Chrome

Arc 自动化需要一个**独立调试 Chrome 实例**，与你日常浏览器隔离。

**Windows 用户：**
双击运行 `launch-chrome-debug.bat`（位于技能目录下），或手动用命令启动：
```bash
"C:\Program Files\Google\Chrome\Application\chrome.exe" ^
  --remote-debugging-port=9333 ^
  --user-data-dir="%USERPROFILE%\.claude\skills\arc-daily-task\chrome-profile" ^
  --no-first-run ^
  --no-default-browser-check ^
  --window-name="Arc House"
```

**macOS 用户：**
```bash
/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome \
  --remote-debugging-port=9333 \
  --user-data-dir="$HOME/.claude/skills/arc-daily-task/chrome-profile" \
  --no-first-run \
  --no-default-browser-check
```

### 3. 配置 chrome-devtools MCP

在 `~/.claude/settings.json` 或 `~/.claude/settings.local.json` 中添加：

```json
{
  "mcpServers": {
    "chrome-devtools": {
      "command": "npx",
      "args": [
        "-y",
        "@anthropic-ai/chrome-devtools-mcp",
        "--browserUrl",
        "http://127.0.0.1:9333"
      ]
    }
  }
}
```

配置完成后需要**重启 Claude Code** 使 MCP 生效。

### 4. 首次登录

1. 启动调试 Chrome
2. 手动访问 `https://community.arc.network/`
3. 完成登录（仅首次需要，登录态持久保存）

## 使用方法

安装配置好后，在 Claude Code 中输入以下任一指令：

```
跑 arc 任务
做 arc 每日积分
arc 签到
arc daily task
```

Claude 会自动执行：检查登录 → 查已做记录 → 阅读 5 篇博客 → 观看 4 个视频 → 生成报告。

## 技能说明

本技能通过 **chrome-devtools MCP** 控制浏览器完成自动化，主要功能：

- **登录检测**：自动检查浏览器是否已登录 Arc Community
- **去重**：读取 my-contributions 页面，自动跳过已做的内容
- **模拟阅读**：打开文章，分 8 段平滑滚动浏览（约 20 秒/篇）
- **视频播放**：通过 CDP 真实鼠标点击播放按钮，播放 40 秒
- **无状态文件**：所有"已做"判定实时来自 Arc 页面，不写本地文件

## 文件说明

| 文件 | 说明 |
|------|------|
| `SKILL.md` | Claude Code 执行指令（含完整中文注释） |
| `launch-chrome-debug.ps1` | PowerShell 启动脚本 |
| `launch-chrome-debug.bat` | Windows 批处理启动脚本 |
| `.gitignore` | Git 忽略规则 |

## 常见问题

| 问题 | 解决 |
|------|------|
| MCP 连接失败 | 启动调试 Chrome（运行 launch-chrome-debug.bat） |
| 阅读没加分 | 确保选了**没读过**的文章，24h 冷却 |
| 视频没加分 | 确保视频实际点击播放了（可能需手动检查） |
| 积分不到账 | 等待约 5 分钟刷新页面确认 |
