# 第 2 章：快速开始

> 本章概述：指导你完成 OpenClaw 的安装、配置和首次使用。通过逐步引导，让你在最短时间内让 OpenClaw 运行起来。

## 学习目标

- 掌握 OpenClaw 的安装方法
- 完成首次配置和网关启动
- 验证安装并发送第一条消息
- 了解后续配置方向

## 前置条件

- Node.js 22 或更高版本
- 一个可用的 LLM API 密钥（OpenAI、Anthropic 或 Google）
- 基础命令行使用经验

---

## 2.1 系统要求

### 2.1.1 硬件要求

| 组件 | 最低要求 | 推荐配置 |
|------|----------|----------|
| CPU | 双核处理器 | 四核或更高 |
| 内存 | 4 GB | 8 GB 或更高 |
| 存储 | 1 GB 可用空间 | 5 GB 或更高 |
| 网络 | 宽带连接 | 稳定的宽带连接 |

### 2.1.2 软件要求

**运行时环境**：
- Node.js 22+（必需）
- npm、pnpm 或 bun（任选其一）

**可选依赖**：
- Docker（用于沙箱隔离）
- Git（用于版本管理）
- Tailscale（用于远程访问）

### 2.1.3 平台支持

| 平台 | 支持状态 | 说明 |
|------|----------|------|
| macOS | 完全支持 | 推荐，包含菜单栏应用 |
| Linux | 完全支持 | systemd 服务管理 |
| Windows (WSL2) | 支持 | 推荐在 WSL2 中运行 |
| Docker | 支持 | 容器化部署 |

<Info>
**检查 Node 版本**：
```bash
node --version
# 应输出 v22.x.x 或更高
```
</Info>

---

## 2.2 安装方式

OpenClaw 提供多种安装方式，选择最适合你的一种：

### 2.2.1 方式一：安装脚本（推荐）

**macOS/Linux**：
```bash
curl -fsSL https://openclaw.ai/install.sh | bash
```

**Windows (PowerShell)**：
```powershell
iwr -useb https://openclaw.ai/install.ps1 | iex
```

安装脚本会自动：
- 检测并安装 Node.js（如未安装）
- 安装 OpenClaw CLI
- 配置基本环境变量

### 2.2.2 方式二：npm/pnpm 安装

```bash
# 使用 npm
npm install -g openclaw@latest

# 使用 pnpm（推荐）
pnpm add -g openclaw@latest

# 使用 bun
bun install -g openclaw@latest
```

### 2.2.3 方式三：Docker 安装

```bash
docker run -d \
  --name openclaw \
  -v ~/.openclaw:/root/.openclaw \
  -p 18789:18789 \
  openclaw/gateway:latest
```

<Note>
**Docker 部署注意**：
- 卷挂载用于持久化配置和数据
- 端口 18789 是 Gateway 默认端口
- 详细配置参考 [Docker 部署指南](https://docs.openclaw.ai/install/docker)
</Note>

### 2.2.4 方式四：源码安装（开发者）

```bash
# 克隆仓库
git clone https://github.com/openclaw/openclaw.git
cd openclaw

# 安装依赖
pnpm install

# 构建
pnpm build

# 运行
pnpm openclaw onboard --install-daemon
```

---

## 2.3 首次配置向导

安装完成后，运行 onboarding 向导完成首次配置：

```bash
openclaw onboard --install-daemon
```

### 2.3.1 向导步骤

向导会引导你完成以下步骤：

<Frame>
```
┌─────────────────────────────────────────────────────────┐
│           OpenClaw Onboarding Wizard                    │
├─────────────────────────────────────────────────────────┤
│  Step 1: 选择 Gateway 运行位置                          │
│  Step 2: 配置 LLM 提供商和 API 密钥                      │
│  Step 3: 设置认证方式                                   │
│  Step 4: 配置消息通道（可选）                           │
│  Step 5: 安装守护进程服务                               │
│  Step 6: 验证安装                                       │
└─────────────────────────────────────────────────────────┘
```
</Frame>

#### Step 1: 选择 Gateway 运行位置

```
? Where should the Gateway run?
❯ This Mac (Local only)
  Remote (over SSH/Tailnet)
  Configure later
```

- **本地运行**：Gateway 和客户端在同一设备
- **远程运行**：Gateway 在远程服务器，通过 SSH/Tailnet 访问

#### Step 2: 配置 LLM 提供商

```
? Select your preferred LLM provider:
❯ Anthropic (Claude)
  OpenAI (GPT)
  Google (Gemini)
  Local (Ollama)
```

<Warning>
**安全提示**：API 密钥会存储在本地凭证存储中（macOS Keychain、Linux Secret Service），不会上传到 OpenClaw 服务器。
</Warning>

#### Step 3: 设置认证方式

```
? Gateway authentication mode:
❯ Token (recommended for remote access)
  Password
  Tailscale only
  None (local only, not recommended)
```

#### Step 4: 配置消息通道（可选）

可以选择在向导中配置消息通道，或稍后手动配置：

```
? Configure a message channel now?
❯ Yes, guide me through channel setup
  No, I'll do it later
  Skip to dashboard
```

#### Step 5: 安装守护进程

```
? Install Gateway as a background service?
❯ Yes (recommended)
  No, run manually
```

选择 "Yes" 后，Gateway 会作为系统服务运行：
- **macOS**: launchd 服务
- **Linux**: systemd 用户服务
- **Windows**: 计划任务（WSL2 环境）

---

## 2.4 验证安装

### 2.4.1 检查 Gateway 状态

```bash
openclaw gateway status
```

期望输出：
```
✓ Gateway is running
  PID: 12345
  Port: 18789
  Auth: token
  Uptime: 2m 34s
```

### 2.4.2 运行健康检查

```bash
openclaw health
```

期望输出：
```
✓ Gateway connection: OK
✓ Authentication: OK
✓ LLM Provider: OK (anthropic/claude-opus-4-6)
✓ Configuration: OK
```

### 2.4.3 打开控制面板

```bash
openclaw dashboard
```

这会在浏览器中打开 Control UI（默认地址：http://127.0.0.1:18789/）

<Frame>
控制面板提供：
- 会话管理
- 消息历史
- 配置编辑
- 日志查看
- 系统监控
</Frame>

### 2.4.4 发送测试消息

**无需通道的测试**（通过 WebChat）：
1. 打开 Dashboard
2. 点击 "WebChat" 标签
3. 输入消息并发送

**有通道后的测试**：
```bash
# 发送消息到指定通道
openclaw message send \
  --target +15555550123 \
  --message "Hello from OpenClaw"
```

### 2.4.5 与 Agent 对话

```bash
openclaw agent --message "Hello, what can you do?"
```

期望输出：
```
[Agent] Hello! I'm OpenClaw, your personal AI assistant. I can:
- Answer questions and have conversations
- Browse the web and retrieve information
- Execute code and run commands
- Control your devices and apps
- Automate repetitive tasks
- And much more!

What would you like me to help you with?
```

---

## 2.5 故障排除

### 问题 1：Gateway 无法启动

**症状**：
```bash
openclaw gateway status
# 输出：Gateway is not running
```

**解决方案**：
```bash
# 检查端口是否被占用
lsof -i :18789

# 手动启动 Gateway
openclaw gateway --port 18789 --verbose

# 查看详细日志
openclaw logs --tail 100
```

### 问题 2：认证失败

**症状**：
```
✗ Authentication failed: invalid token
```

**解决方案**：
```bash
# 重新生成 token
openclaw gateway reset-auth

# 或重新运行向导
openclaw onboard
```

### 问题 3：LLM 连接失败

**症状**：
```
✗ LLM Provider connection failed
```

**解决方案**：
1. 检查 API 密钥是否正确
2. 检查网络连接
3. 尝试切换到其他提供商：
```bash
openclaw models use openai/gpt-4o
```

### 问题 4：权限问题（macOS）

**症状**：
```
Permission denied: cannot access screen recording
```

**解决方案**：
1. 打开 系统设置 → 隐私与安全性
2. 授予 OpenClaw 以下权限：
   - 屏幕录制
   - 辅助功能
   - 自动化
   - 麦克风
   - 摄像头

---

## 2.6 下一步

安装完成后，你可以：

| 方向 | 文档 |
|------|------|
| 配置消息通道 | [第 5 章：消息通道配置](chapter-05.md) |
| 了解核心概念 | [第 3 章：核心概念](chapter-03.md) |
| 配置网关 | [第 4 章：网关配置详解](chapter-04.md) |
| 设置安全权限 | [第 7 章：安全与权限](chapter-07.md) |

### 推荐阅读路径

**初学者**：
```
第 2 章（本章）→ 第 3 章 → 第 4 章 → 第 5 章
```

**开发者**：
```
第 2 章（本章）→ 第 3 章 → 第 8 章 → 第 10 章
```

**运维人员**：
```
第 2 章（本章）→ 第 4 章 → 第 16 章 → 第 18 章
```

---

## 本章小结

- **安装方式**：推荐使用安装脚本或 npm/pnpm 全局安装
- **首次配置**：运行 `openclaw onboard --install-daemon` 完成向导
- **验证安装**：使用 `openclaw health` 和 `openclaw dashboard` 检查状态
- **故障排除**：查看日志 `openclaw logs` 和运行 `openclaw doctor`

## 延伸阅读

- [onboarding 向导详解](https://docs.openclaw.ai/start/wizard)
- [Gateway 运行手册](https://docs.openclaw.ai/gateway)
- [控制面板使用](https://docs.openclaw.ai/web/control-ui)
- [第 3 章：核心概念](chapter-03.md)

---

*上一章：[第 1 章：认识 OpenClaw](chapter-01.md) | 下一章：[第 3 章：核心概念](chapter-03.md)*
