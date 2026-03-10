# 附录 A：CLI 命令参考

> 本附录提供 OpenClaw 命令行接口（CLI）的完整参考，包括所有命令、选项和使用示例。

## A.1 命令分类

### A.1.1 核心命令

| 命令 | 说明 | 用法 |
|------|------|------|
| `openclaw` | 主命令入口 | `openclaw <command> [options]` |
| `openclaw help` | 显示帮助信息 | `openclaw help [command]` |
| `openclaw version` | 显示版本信息 | `openclaw version` |

### A.1.2 Gateway 管理

| 命令 | 说明 | 用法 |
|------|------|------|
| `openclaw gateway` | Gateway 管理主命令 | `openclaw gateway <subcommand>` |
| `openclaw gateway start` | 启动 Gateway | `openclaw gateway start` |
| `openclaw gateway stop` | 停止 Gateway | `openclaw gateway stop` |
| `openclaw gateway restart` | 重启 Gateway | `openclaw gateway restart` |
| `openclaw gateway status` | 查看 Gateway 状态 | `openclaw gateway status` |
| `openclaw gateway probe` | 探测 Gateway 健康 | `openclaw gateway probe` |
| `openclaw gateway install` | 安装为系统服务 | `openclaw gateway install` |
| `openclaw gateway uninstall` | 卸载系统服务 | `openclaw gateway uninstall` |

**Gateway 选项**：

```bash
openclaw gateway start \
  --port 18789 \
  --bind loopback \
  --verbose \
  --no-daemon
```

| 选项 | 说明 | 默认值 |
|------|------|--------|
| `--port <number>` | Gateway 端口 | `18789` |
| `--bind <interface>` | 绑定接口（loopback/lan/tailnet） | `loopback` |
| `--verbose` | 详细日志 | `false` |
| `--no-daemon` | 前台运行 | `false` |
| `--token <token>` | 认证 token | - |
| `--password <password>` | 认证密码 | - |

### A.1.3 状态和诊断

| 命令 | 说明 | 用法 |
|------|------|------|
| `openclaw status` | 查看系统状态 | `openclaw status [--all]` |
| `openclaw doctor` | 健康检查和修复 | `openclaw doctor [--repair]` |
| `openclaw health` | 健康检查 | `openclaw health` |
| `openclaw logs` | 查看日志 | `openclaw logs [--follow]` |

**Status 选项**：

```bash
openclaw status --all --json
```

| 选项 | 说明 | 默认值 |
|------|------|--------|
| `--all` | 完整报告 | `false` |
| `--json` | JSON 输出 | `false` |
| `--deep` | 深度检查 | `false` |

**Doctor 选项**：

```bash
openclaw doctor --repair --yes
```

| 选项 | 说明 | 默认值 |
|------|------|--------|
| `--yes` | 接受默认修复 | `false` |
| `--repair` | 应用修复 | `false` |
| `--force` | 强制修复 | `false` |
| `--non-interactive` | 非交互模式 | `false` |
| `--deep` | 深度扫描 | `false` |

**Logs 选项**：

```bash
openclaw logs --follow --limit 100 --level error
```

| 选项 | 说明 | 默认值 |
|------|------|--------|
| `--follow` | 实时跟踪 | `false` |
| `--limit <n>` | 行数限制 | `100` |
| `--level <level>` | 日志级别 | `info` |
| `--since <time>` | 起始时间 | - |
| `--until <time>` | 结束时间 | - |

### A.1.4 通道管理

| 命令 | 说明 | 用法 |
|------|------|------|
| `openclaw channels` | 通道管理主命令 | `openclaw channels <subcommand>` |
| `openclaw channels status` | 查看通道状态 | `openclaw channels status [--probe]` |
| `openclaw channels login` | 登录通道（如 WhatsApp） | `openclaw channels login --channel whatsapp` |
| `openclaw channels logout` | 登出通道 | `openclaw channels logout --channel <channel>` |

**Channels Status 选项**：

```bash
openclaw channels status --probe --json
```

| 选项 | 说明 | 默认值 |
|------|------|--------|
| `--probe` | 主动探测 | `false` |
| `--json` | JSON 输出 | `false` |
| `--channel <channel>` | 指定通道 | - |

**Channels Login 选项**：

```bash
openclaw channels login \
  --channel whatsapp \
  --verbose \
  --qr
```

| 选项 | 说明 | 默认值 |
|------|------|--------|
| `--channel <channel>` | 通道名称 | - |
| `--verbose` | 详细输出 | `false` |
| `--qr` | 显示 QR 码 | `false` |

### A.1.5 设备配对

| 命令 | 说明 | 用法 |
|------|------|------|
| `openclaw devices` | 设备管理主命令 | `openclaw devices <subcommand>` |
| `openclaw devices list` | 列出设备请求 | `openclaw devices list` |
| `openclaw devices approve` | 批准设备 | `openclaw devices approve <requestId>` |
| `openclaw devices reject` | 拒绝设备 | `openclaw devices reject <requestId>` |
| `openclaw devices approved` | 列出已批准设备 | `openclaw devices approved` |
| `openclaw devices remove` | 移除设备 | `openclaw devices remove <deviceId>` |

**Approve 选项**：

```bash
openclaw devices approve <requestId> --channel whatsapp
```

| 选项 | 说明 | 默认值 |
|------|------|--------|
| `--channel <channel>` | 指定通道 | - |
| `--account <id>` | 多账号通道 | - |

### A.1.6 Agent 管理

| 命令 | 说明 | 用法 |
|------|------|------|
| `openclaw agents` | Agent 管理主命令 | `openclaw agents <subcommand>` |
| `openclaw agents list` | 列出 Agent | `openclaw agents list` |
| `openclaw agents status` | Agent 状态 | `openclaw agents status` |
| `openclaw agent` | 单 Agent 操作 | `openclaw agent <subcommand>` |

### A.1.7 会话管理

| 命令 | 说明 | 用法 |
|------|------|------|
| `openclaw sessions` | 会话管理主命令 | `openclaw sessions <subcommand>` |
| `openclaw sessions list` | 列出会话 | `openclaw sessions list` |
| `openclaw sessions describe` | 描述会话 | `openclaw sessions describe <session-key>` |
| `openclaw sessions remove` | 删除会话 | `openclaw sessions remove <session-key>` |
| `openclaw sessions compact` | 压缩会话 | `openclaw sessions compact <session-key>` |

**Sessions List 选项**：

```bash
openclaw sessions list --limit 20 --json
```

| 选项 | 说明 | 默认值 |
|------|------|--------|
| `--limit <n>` | 限制数量 | `20` |
| `--json` | JSON 输出 | `false` |
| `--active` | 仅活跃会话 | `false` |

### A.1.8 Cron 调度

| 命令 | 说明 | 用法 |
|------|------|------|
| `openclaw cron` | Cron 管理主命令 | `openclaw cron <subcommand>` |
| `openclaw cron list` | 列出 Cron 任务 | `openclaw cron list` |
| `openclaw cron status` | Cron 状态 | `openclaw cron status` |
| `openclaw cron add` | 添加任务 | `openclaw cron add [options]` |
| `openclaw cron edit` | 编辑任务 | `openclaw cron edit <job-id>` |
| `openclaw cron run` | 运行任务 | `openclaw cron run <job-id>` |
| `openclaw cron remove` | 删除任务 | `openclaw cron remove <job-id>` |
| `openclaw cron enable` | 启用任务 | `openclaw cron enable <job-id>` |
| `openclaw cron disable` | 禁用任务 | `openclaw cron disable <job-id>` |
| `openclaw cron runs` | 运行历史 | `openclaw cron runs --id <job-id>` |

**Cron Add 选项**：

```bash
openclaw cron add \
  --name "Morning brief" \
  --cron "0 7 * * *" \
  --tz "Asia/Shanghai" \
  --session isolated \
  --message "Summarize overnight updates." \
  --announce \
  --channel whatsapp \
  --to "+8613800138000"
```

| 选项 | 说明 | 默认值 |
|------|------|--------|
| `--name <name>` | 任务名称 | - |
| `--cron <expr>` | Cron 表达式 | - |
| `--at <time>` | 一次性时间 | - |
| `--every <duration>` | 固定间隔 | - |
| `--tz <timezone>` | 时区 | 主机时区 |
| `--session <type>` | 会话类型（main/isolated） | `main` |
| `--message <text>` | Agent 消息 | - |
| `--system-event <text>` | 系统事件文本 | - |
| `--model <model>` | 模型覆盖 | - |
| `--thinking <level>` | 思考级别 | - |
| `--announce` | Announce 投递 | `false` |
| `--channel <channel>` | 投递通道 | - |
| `--to <recipient>` | 投递目标 | - |
| `--wake <mode>` | 唤醒模式（now/next-heartbeat） | `now` |
| `--delete-after-run` | 运行后删除 | `false`（周期性）/`true`（一次性） |
| `--exact` | 禁用错开 | `false` |
| `--stagger <duration>` | 错开窗口 | - |
| `--light-context` | 轻量级上下文 | `false` |
| `--agent <agent-id>` | 指定 Agent | - |

**Cron Edit 选项**：

```bash
openclaw cron edit <job-id> \
  --message "Updated prompt" \
  --model opus \
  --thinking low
```

| 选项 | 说明 | 默认值 |
|------|------|--------|
| `--name <name>` | 任务名称 | - |
| `--message <text>` | Agent 消息 | - |
| `--model <model>` | 模型覆盖 | - |
| `--thinking <level>` | 思考级别 | - |
| `--enabled` | 启用 | - |
| `--disabled` | 禁用 | - |
| `--agent <agent-id>` | Agent 绑定 | - |
| `--clear-agent` | 清除 Agent | - |

**Cron Run 选项**：

```bash
openclaw cron run <job-id> --due
```

| 选项 | 说明 | 默认值 |
|------|------|--------|
| `--due` | 仅到期时运行 | `false`（默认强制） |

**Cron Runs 选项**：

```bash
openclaw cron runs --id <job-id> --limit 50
```

| 选项 | 说明 | 默认值 |
|------|------|--------|
| `--id <job-id>` | 任务 ID | - |
| `--limit <n>` | 限制数量 | `50` |

### A.1.9 节点管理

| 命令 | 说明 | 用法 |
|------|------|------|
| `openclaw nodes` | 节点管理主命令 | `openclaw nodes <subcommand>` |
| `openclaw nodes list` | 列出节点 | `openclaw nodes list` |
| `openclaw nodes status` | 节点状态 | `openclaw nodes status` |
| `openclaw nodes describe` | 描述节点 | `openclaw nodes describe --node <id>` |
| `openclaw nodes invoke` | 调用节点命令 | `openclaw nodes invoke --node <id> --command <cmd>` |
| `openclaw nodes run` | 运行系统命令 | `openclaw nodes run --node <id> --raw <cmd>` |
| `openclaw nodes notify` | 发送通知 | `openclaw nodes notify --node <id> --title <title>` |

**Nodes Invoke 选项**：

```bash
openclaw nodes invoke \
  --node "iOS Node" \
  --command canvas.navigate \
  --params '{"url":"http://localhost:18789/__openclaw__/canvas/"}'
```

| 选项 | 说明 | 默认值 |
|------|------|--------|
| `--node <id|name|ip>` | 节点标识 | - |
| `--command <command>` | 命令名称 | - |
| `--params <json>` | JSON 参数 | `{}` |
| `--invoke-timeout <ms>` | 调用超时 | `15000` |
| `--idempotency-key <key>` | 幂等键 | - |

**Nodes Run 选项**：

```bash
openclaw nodes run \
  --agent main \
  --node "macOS Node" \
  --raw "git status" \
  --cwd /path/to/work
```

| 选项 | 说明 | 默认值 |
|------|------|--------|
| `--agent <agent-id>` | 审批 Agent | - |
| `--node <id>` | 节点 ID | - |
| `--raw <command>` | 原始命令 | - |
| `--cwd <path>` | 工作目录 | - |
| `--env <key=val>` | 环境变量 | - |
| `--command-timeout <ms>` | 命令超时 | - |
| `--needs-screen-recording` | 需要屏幕录制 | `false` |

**Nodes Notify 选项**：

```bash
openclaw nodes notify \
  --node "macOS Node" \
  --title "提醒" \
  --body "会议即将开始" \
  --sound Glass
```

| 选项 | 说明 | 默认值 |
|------|------|--------|
| `--node <id>` | 节点 ID | - |
| `--title <title>` | 通知标题 | - |
| `--body <body>` | 通知内容 | - |
| `--sound <sound>` | 通知声音 | - |

### A.1.10 Hooks 管理

| 命令 | 说明 | 用法 |
|------|------|------|
| `openclaw hooks` | Hooks 管理主命令 | `openclaw hooks <subcommand>` |
| `openclaw hooks list` | 列出 Hooks | `openclaw hooks list` |
| `openclaw hooks info` | Hook 详情 | `openclaw hooks info <hook-name>` |
| `openclaw hooks check` | 检查资格 | `openclaw hooks check` |
| `openclaw hooks enable` | 启用 Hook | `openclaw hooks enable <hook-name>` |
| `openclaw hooks disable` | 禁用 Hook | `openclaw hooks disable <hook-name>` |
| `openclaw hooks install` | 安装 Hook Pack | `openclaw hooks install <spec>` |
| `openclaw hooks uninstall` | 卸载 Hook Pack | `openclaw hooks uninstall <hook-name>` |

**Hooks List 选项**：

```bash
openclaw hooks list --verbose --eligible --json
```

| 选项 | 说明 | 默认值 |
|------|------|--------|
| `--verbose` | 详细输出（显示缺失要求） | `false` |
| `--eligible` | 仅合格 Hooks | `false` |
| `--json` | JSON 输出 | `false` |

**Hooks Info 选项**：

```bash
openclaw hooks info session-memory --json
```

| 选项 | 说明 | 默认值 |
|------|------|--------|
| `--json` | JSON 输出 | `false` |

### A.1.11 配置管理

| 命令 | 说明 | 用法 |
|------|------|------|
| `openclaw config` | 配置管理主命令 | `openclaw config <subcommand>` |
| `openclaw config get` | 获取配置 | `openclaw config get <key>` |
| `openclaw config set` | 设置配置 | `openclaw config set <key> <value>` |
| `openclaw config export` | 导出配置 | `openclaw config export` |
| `openclaw config import` | 导入配置 | `openclaw config import <file>` |

**Config Get 示例**：

```bash
openclaw config get gateway
openclaw config get agents.defaults.heartbeat
openclaw config get channels.telegram
```

**Config Set 示例**：

```bash
openclaw config set gateway.auth.token "new-token"
openclaw config set agents.defaults.heartbeat.every "60m"
```

### A.1.12 浏览器管理

| 命令 | 说明 | 用法 |
|------|------|------|
| `openclaw browser` | 浏览器管理主命令 | `openclaw browser <subcommand>` |
| `openclaw browser status` | 浏览器状态 | `openclaw browser status` |
| `openclaw browser start` | 启动浏览器 | `openclaw browser start` |
| `openclaw browser stop` | 停止浏览器 | `openclaw browser stop` |

### A.1.13 技能和插件

| 命令 | 说明 | 用法 |
|------|------|------|
| `openclaw skills` | 技能管理主命令 | `openclaw skills <subcommand>` |
| `openclaw skills list` | 列出技能 | `openclaw skills list` |
| `openclaw clawhub` | ClawHub 包管理 | `openclaw clawhub <subcommand>` |
| `openclaw clawhub search` | 搜索技能 | `openclaw clawhub search <query>` |
| `openclaw clawhub install` | 安装技能 | `openclaw clawhub install <package>` |
| `openclaw clawhub update` | 更新技能 | `openclaw clawhub update [package]` |
| `openclaw clawhub uninstall` | 卸载技能 | `openclaw clawhub uninstall <package>` |
| `openclaw plugins` | 插件管理主命令 | `openclaw plugins <subcommand>` |
| `openclaw plugins list` | 列出插件 | `openclaw plugins list` |
| `openclaw plugins install` | 安装插件 | `openclaw plugins install <spec>` |
| `openclaw plugins uninstall` | 卸载插件 | `openclaw plugins uninstall <name>` |

**ClawHub 选项**：

```bash
openclaw clawhub install @openclaw/pdf-editor
openclaw clawhub update --all
openclaw clawhub sync --all
```

### A.1.14 系统命令

| 命令 | 说明 | 用法 |
|------|------|------|
| `openclaw system` | 系统命令主命令 | `openclaw system <subcommand>` |
| `openclaw system event` | 触发系统事件 | `openclaw system event --text "..." --mode now` |
| `openclaw system heartbeat` | 心跳控制 | `openclaw system heartbeat last` |

**System Event 选项**：

```bash
openclaw system event \
  --text "检查紧急待办" \
  --mode now
```

| 选项 | 说明 | 默认值 |
|------|------|--------|
| `--text <text>` | 事件文本 | - |
| `--mode <mode>` | 模式（now/next-heartbeat） | `now` |

### A.1.15 其他命令

| 命令 | 说明 | 用法 |
|------|------|------|
| `openclaw onboarding` | 新手引导 | `openclaw onboarding` |
| `openclaw setup` | 安装设置 | `openclaw setup` |
| `openclaw update` | 更新 OpenClaw | `openclaw update` |
| `openclaw uninstall` | 卸载 OpenClaw | `openclaw uninstall` |
| `openclaw backup` | 备份数据 | `openclaw backup` |
| `openclaw reset` | 重置状态 | `openclaw reset` |
| `openclaw tui` | 文本用户界面 | `openclaw tui` |
| `openclaw dashboard` | 打开 Dashboard | `openclaw dashboard` |
| `openclaw completion` | Shell 补全 | `openclaw completion bash` |

---

## A.2 环境变量

### A.2.1 Gateway 配置

| 变量 | 说明 | 默认值 |
|------|------|--------|
| `OPENCLAW_CONFIG_DIR` | 配置目录 | `~/.openclaw` |
| `OPENCLAW_STATE_DIR` | 状态目录 | `~/.openclaw` |
| `OPENCLAW_WORKSPACE_DIR` | 工作区目录 | `~/.openclaw/workspace` |
| `OPENCLAW_GATEWAY_PORT` | Gateway 端口 | `18789` |
| `OPENCLAW_GATEWAY_BIND` | 绑定接口 | `loopback` |
| `OPENCLAW_GATEWAY_TOKEN` | 认证 token | - |
| `OPENCLAW_GATEWAY_PASSWORD` | 认证密码 | - |
| `OPENCLAW_GATEWAY_MODE` | Gateway 模式 | `local` |

### A.2.2 Docker 部署

| 变量 | 说明 | 默认值 |
|------|------|--------|
| `OPENCLAW_IMAGE` | Docker 镜像 | `openclaw:local` |
| `OPENCLAW_DOCKER_APT_PACKAGES` | 额外 apt 包 | - |
| `OPENCLAW_EXTENSIONS` | 预安装扩展 | - |
| `OPENCLAW_EXTRA_MOUNTS` | 额外挂载 | - |
| `OPENCLAW_HOME_VOLUME` | 持久化 /home/node | - |
| `OPENCLAW_SANDBOX` | 启用沙箱 | `0` |

### A.2.3 调试和日志

| 变量 | 说明 | 默认值 |
|------|------|--------|
| `DEBUG` | 调试命名空间 | - |
| `DEBUG_COLORS` | 调试颜色 | `1` |
| `LOG_LEVEL` | 日志级别 | `info` |
| `OPENCLAW_VERBOSE` | 详细日志 | `0` |

### A.2.4 其他

| 变量 | 说明 | 默认值 |
|------|------|--------|
| `OPENCLAW_SKIP_CRON` | 跳过 Cron | `0` |
| `OPENCLAW_ALLOW_INSECURE_PRIVATE_WS` | 允许不安全 WebSocket | `0` |
| `OPENCLAW_PROFILE` | 配置名称 | - |

---

## A.3 配置文件位置

### A.3.1 默认位置

| 文件 | 位置 | 说明 |
|------|------|------|
| 主配置 | `~/.openclaw/openclaw.json` | 主要配置文件 |
| 会话存储 | `~/.openclaw/agents/<agent-id>/sessions/` | 会话数据 |
| Agent 目录 | `~/.openclaw/agents/<agent-id>/agent/` | Agent 工作区 |
| 凭证 | `~/.openclaw/credentials/` | 认证凭证 |
| 日志 | `~/.openclaw/logs/` | 日志文件 |
| Hooks | `~/.openclaw/hooks/` | 托管 Hooks |
| Cron | `~/.openclaw/cron/jobs.json` | Cron 任务 |

### A.3.2 覆盖位置

```bash
# 使用自定义配置目录
export OPENCLAW_CONFIG_DIR=/path/to/config

# 使用自定义状态目录
export OPENCLAW_STATE_DIR=/path/to/state

# 使用自定义工作区
export OPENCLAW_WORKSPACE_DIR=/path/to/workspace
```

---

## A.4 Shell 补全

### A.4.1 Bash

```bash
# 添加到 ~/.bashrc
openclaw completion bash >> ~/.bash_completion
source ~/.bash_completion
```

### A.4.2 Zsh

```bash
# 添加到 ~/.zshrc
openclaw completion zsh >> ~/.zsh/_openclaw
fpath=(~/.zsh $fpath)
autoload -Uz compinit && compinit
```

### A.4.3 Fish

```fish
# 生成补全文件
openclaw completion fish > ~/.config/fish/completions/openclaw.fish
```

---

## A.5 常见问题

### A.5.1 CLI 未找到

```bash
# 检查安装
which openclaw

# 重新安装
pnpm install -g openclaw

# 或从源码
cd /path/to/openclaw
pnpm install
pnpm build
```

### A.5.2 权限错误

```bash
# macOS/Linux
sudo chown -R $(whoami) ~/.openclaw

# Windows（管理员 PowerShell）
icacls "$env:USERPROFILE\.openclaw" /grant "$env:USERNAME:(OI)(CI)F" /T
```

### A.5.3 配置验证

```bash
# 导出并检查配置
openclaw config export > config.json
cat config.json | jq .
```

---

*返回：[第 18 章：故障排除](chapter-18.md) | 下一章：[附录 B：配置 Schema](appendix-b.md)*
