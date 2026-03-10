# OpenClaw 设计原理与使用手册

## 目录

1. [项目概述](#项目概述)
2. [设计原理](#设计原理)
3. [系统架构](#系统架构)
4. [核心机制](#核心机制)
5. [安装与配置](#安装与配置)
6. [使用指南](#使用指南)
7. [插件开发](#插件开发)
8. [故障排查](#故障排查)

---

## 项目概述

### 什么是 OpenClaw

OpenClaw 是一个**个人 AI 助手**，运行在您自己的设备上，通过您已有的通讯渠道（WhatsApp、Telegram、Slack、Discord、Signal、iMessage 等）进行交互。

**核心理念**：
- **本地优先**：Gateway 运行在本地，数据本地存储
- **多通道支持**：统一的 WebSocket 控制平面连接多个消息渠道
- **隐私安全**：敏感信息本地处理，安全默认配置
- **可扩展**：插件系统支持自定义功能

### 技术栈

- **运行时**：Node.js 22+
- **语言**：TypeScript (ESM)
- **包管理**：pnpm / Bun
- **测试框架**：Vitest
- **UI 框架**：Lit (Web UI), SwiftUI (macOS/iOS)

### 版本信息

- 当前版本：2026.3.9
- 发布频道：stable / beta / dev
- 许可证：MIT

---

## 设计原理

### 1. Gateway 控制平面架构

OpenClaw 采用单一 Gateway 控制平面设计：

```
┌─────────────────────────────────────────────────────────────┐
│                    消息渠道层 (Channels)                      │
│  WhatsApp │ Telegram │ Slack │ Discord │ Signal │ iMessage  │
│  IRC      │ Teams    │ Matrix │ LINE   │ Zalo   │ Nostr     │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Gateway (控制平面)                         │
│  ┌─────────────┐ ┌──────────────┐ ┌─────────────────────┐   │
│  │ WebSocket   │ │ 会话管理     │ │ 工具执行引擎        │   │
│  │ API Server  │ │ Session Mgr  │ │ Tool Executor       │   │
│  └─────────────┘ └──────────────┘ └─────────────────────┘   │
│  ┌─────────────┐ ┌──────────────┐ ┌─────────────────────┐   │
│  │ 插件运行时  │ │ 配置管理     │ │ 认证与授权          │   │
│  │ Plugin Rtm  │ │ Config Mgr   │ │ Auth & Policy       │   │
│  └─────────────┘ └──────────────┘ └─────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
        ┌──────────┐   ┌──────────┐   ┌──────────┐
        │   CLI    │   │  Web UI  │   │  macOS   │
        │  客户端  │   │  控制台  │   │   App    │
        └──────────┘   └──────────┘   └──────────┘
```

**设计优势**：
- 单一控制点管理所有消息表面
- 客户端（CLI、Web、App）通过统一 WebSocket 协议连接
- 插件在进程内运行，可访问完整 Gateway 能力

### 2. 会话隔离机制

每个消息会话被映射到独立的 Agent 会话：

- **会话键生成**：基于渠道类型 + 账号 ID + 对话者 ID 派生唯一会话键
- **上下文隔离**：不同会话的上下文独立管理，避免信息泄露
- **会话持久化**：会话历史存储在本地 SQLite 数据库

```typescript
// 会话键派生逻辑
const sessionKey = deriveSessionKey({
  channelType: 'whatsapp',
  accountId: 'default',
  peerId: '+1234567890'
});
```

### 3. 安全设计原则

**DM（私聊）安全策略**：

| 策略 | 行为 | 适用场景 |
|------|------|----------|
| `pairing`（默认） | 未知发送者收到配对码，需管理员批准 | 公开部署 |
| `open` | 允许所有消息，需显式配置 `*` 白名单 | 私有/受信环境 |

**设备配对流程**：
1. 新设备连接时提交身份声明
2. Gateway 生成配对码并发送给管理员渠道
3. 管理员批准后将设备加入本地信任存储
4. 设备获得持久化令牌用于后续连接

**认证层级**：
```
本地连接 (loopback) → 自动批准（可选密码）
    ↓
尾网连接 (Tailscale) → 令牌认证
    ↓
公网连接 (SSH 隧道) → 令牌 + 密码双重认证
```

### 4. 插件系统设计

**插件类型**：

| 类型 | 功能 | 注册 API |
|------|------|----------|
| 渠道插件 | 添加新消息渠道 | `api.registerChannel()` |
| 工具插件 | 添加 Agent 工具 | `api.registerTool()` |
| 命令插件 | 添加 CLI 命令 | `api.registerCli()` |
| 服务插件 | 后台服务 | `api.registerService()` |
| 提供者插件 | 模型认证 | `api.registerProvider()` |

**加载顺序**（优先级从高到低）：
1. 配置路径 (`plugins.load.paths`)
2. 工作区扩展 (`<workspace>/.openclaw/extensions/`)
3. 全局扩展 (`~/.openclaw/extensions/`)
4. 捆绑扩展 (`<openclaw>/extensions/`)

**安全机制**：
- 插件路径符号链接解析后必须在插件根目录内
- 世界可写目录被拒绝
- 非捆绑插件的所有权被检查

---

## 系统架构

### 核心组件

#### 1. Gateway Server

**位置**：`src/gateway/gateway.ts`

**职责**：
- WebSocket 服务器（默认端口 18789）
- 管理所有渠道连接
- 处理客户端请求和事件推送
- 运行插件运行时

**关键方法**：
```typescript
// 启动 Gateway
await gateway.start({
  bind: '127.0.0.1',
  port: 18789,
  auth: { mode: 'token', token: '...' }
});

// 注册渠道
await gateway.registerChannel({ id: 'telegram', config: {...} });

// 发送消息
await gateway.send({
  channel: 'telegram:default',
  to: '+1234567890',
  text: 'Hello'
});
```

#### 2. Agent 运行时

**位置**：`src/agents/`

**职责**：
- 执行 Agent 循环（接收消息 → 构建提示 → 调用模型 → 处理响应）
- 管理工具执行
- 处理会话上下文

**Agent 循环流程**：
```
1. 接收消息 (从渠道)
        ↓
2. 加载会话 (SQLite)
        ↓
3. 构建提示 (系统提示 + 上下文 + 用户消息)
        ↓
4. 调用模型 (支持多个提供者)
        ↓
5. 流式响应 (边生成边发送)
        ↓
6. 更新上下文 (存储历史)
```

#### 3. 渠道层

**位置**：`src/telegram/`, `src/discord/`, `src/whatsapp/` 等

**统一接口**：
```typescript
interface Channel {
  // 启动渠道连接
  start(): Promise<void>;

  // 停止渠道
  stop(): Promise<void>;

  // 发送消息
  sendText(params: SendParams): Promise<SendResult>;

  // 获取账号状态
  getStatus(): Promise<ChannelStatus>;
}
```

**支持的渠道**：

| 渠道 | 实现 | 状态 |
|------|------|------|
| WhatsApp | Baileys | 内置 |
| Telegram | grammY | 内置 |
| Slack | Bolt | 内置 |
| Discord | discord.js | 内置 |
| Signal | signal-cli | 插件 |
| iMessage | BlueBubbles | 插件 |
| LINE | LINE SDK | 插件 |
| Matrix | matrix-js-sdk | 插件 |
| Nostr | nostr-tools | 插件 |
| Zalo | 自定义 | 插件 |

#### 4. 配置系统

**配置层次**：

```
1. 环境变量 (最高优先级)
        ↓
2. 工作区配置 (<workspace>/.openclaw/config.json)
        ↓
3. 全局配置 (~/.openclaw/config.json)
        ↓
4. 默认配置 (最低优先级)
```

**关键配置项**：

```json5
{
  // Gateway 配置
  gateway: {
    bind: '127.0.0.1',      // 绑定地址
    port: 18789,            // 端口
    auth: {
      mode: 'token',        // 认证模式
      token: 'your-token'   // 令牌
    }
  },

  // 模型配置
  models: {
    default: 'anthropic/claude-opus-4-6',
    providers: {
      anthropic: { apiKey: '...' },
      openai: { apiKey: '...' }
    }
  },

  // 渠道配置
  channels: {
    telegram: {
      accounts: {
        default: {
          token: 'BOT_TOKEN',
          enabled: true
        }
      }
    }
  },

  // 插件配置
  plugins: {
    enabled: true,
    slots: {
      memory: 'memory-core'
    },
    entries: {
      'voice-call': {
        enabled: true,
        config: { provider: 'twilio' }
      }
    }
  }
}
```

---

## 核心机制

### 1. WebSocket 协议

**协议版本**：v3

**连接握手**：
```typescript
// 客户端发送
{
  type: 'req',
  method: 'connect',
  id: 'uuid-1',
  params: {
    device: {
      id: 'device-uuid',
      platform: 'macos',
      name: 'My Mac'
    },
    auth: {
      token: 'gateway-token'
    }
  }
}

// Gateway 响应
{
  type: 'res',
  id: 'uuid-1',
  ok: true,
  payload: {
    type: 'hello-ok',
    snapshot: {
      presence: {...},
      health: {...}
    }
  }
}
```

**请求/响应模式**：
```typescript
// 请求格式
{
  type: 'req',
  id: 'unique-id',
  method: 'agent',
  params: { message: 'Hello', thinking: 'high' }
}

// 响应格式
{
  type: 'res',
  id: 'unique-id',
  ok: true,
  payload: { runId: '...', status: 'accepted' }
}
```

**事件推送**：
```typescript
// Agent 事件（流式）
{
  type: 'event',
  event: 'agent',
  payload: {
    runId: '...',
    chunk: { type: 'text', text: 'Hello...' }
  }
}

// 状态事件
{
  type: 'event',
  event: 'presence',
  payload: { online: true, channel: 'telegram' }
}
```

### 2. 认证与配对

**配对流程**：

```
1. 新设备连接
        ↓
2. Gateway 生成 6 位配对码
        ↓
3. 配对码发送到管理员渠道
        ↓
4. 管理员在渠道内批准
        ↓
5. 设备获得持久化令牌
        ↓
6. 后续连接使用令牌认证
```

**CLI 批准命令**：
```bash
# 查看待批准的配对请求
openclaw pairing pending

# 批准配对
openclaw pairing approve telegram <code>

# 拒绝配对
openclaw pairing reject telegram <code>
```

### 3. 工具执行机制

**内置工具**：

| 工具 | 功能 | 权限 |
|------|------|------|
| `bash` | 执行 shell 命令 | 需批准 |
| `browser` | 浏览器控制 | 默认允许 |
| `canvas` | Canvas 操作 | 默认允许 |
| `node.invoke` | 调用设备节点 | 需配对 |

**工具执行流程**：
```
1. Agent 决定调用工具
        ↓
2. Gateway 检查工具权限
        ↓
3. 如需批准 → 发送批准请求给管理员
        ↓
4. 管理员批准 → 执行工具
        ↓
5. 返回结果给 Agent
```

**批准策略**：
```json5
{
  tools: {
    bash: {
      approval: 'manual',     // manual/auto/never
      allowPatterns: ['ls', 'cat', 'grep'],
      denyPatterns: ['rm -rf', 'sudo']
    }
  }
}
```

### 4. 上下文管理

**上下文引擎**：

```typescript
interface ContextEngine {
  // 摄取新消息
  ingest(message: Message): Promise<void>;

  // 组装提示
  assemble(params: AssembleParams): Promise<Prompt>;

  // 压缩上下文
  compact(): Promise<CompactResult>;
}
```

**默认行为**：
- 保留最近的 N 条消息（可配置）
- 超过阈值时自动压缩
- 压缩使用 LLM 生成摘要

### 5. 模型提供者

**支持的提供者**：

| 提供者 | 认证方式 | 配置 |
|--------|----------|------|
| Anthropic | API Key | `models.providers.anthropic.apiKey` |
| OpenAI | API Key / OAuth | `models.providers.openai.apiKey` |
| Google | API Key / OAuth | `models.providers.google.credentials` |
| BytePlus | API Key | `models.providers.byteplus.apiKey` |
| AWS Bedrock | IAM | `models.providers.bedrock.credentials` |

**模型故障转移**：
```json5
{
  models: {
    default: 'anthropic/claude-opus-4-6',
    failover: [
      'openai/gpt-5.2',
      'google/gemini-2.5-pro'
    ]
  }
}
```

---

## 安装与配置

### 系统要求

- Node.js 22.12.0+
- pnpm 10.23.0+（推荐）或 Bun
- 操作系统：macOS 14+ / Linux / Windows (WSL2 推荐)

### 快速安装

```bash
# 全局安装
npm install -g openclaw@latest
# 或
pnpm add -g openclaw@latest

# 运行向导
openclaw onboard --install-daemon
```

### 从源码安装

```bash
# 克隆仓库
git clone https://github.com/openclaw/openclaw.git
cd openclaw

# 安装依赖
pnpm install

# 构建
pnpm build

# 运行向导
pnpm openclaw onboard --install-daemon

# 开发模式
pnpm gateway:watch
```

### 配置步骤

#### 1. 设置模型提供者

```bash
# Anthropic
openclaw models auth login --provider anthropic
# 或直接设置 API Key
openclaw config set models.providers.anthropic.apiKey sk-ant-xxx

# OpenAI
openclaw models auth login --provider openai

# Google (OAuth)
openclaw models auth login --provider google --method oauth
```

#### 2. 配置消息渠道

```bash
# 配置 Telegram
openclaw channels configure telegram
# 按提示输入 Bot Token

# 配置 WhatsApp
openclaw channels configure whatsapp
# 扫描二维码配对

# 配置 Signal
openclaw plugins install @openclaw/signal
openclaw channels configure signal
```

#### 3. 设置认证

```bash
# 生成网关令牌
openclaw config set gateway.auth.token $(openssl rand -hex 32)

# 启用 Tailscale 集成（可选）
openclaw config set gateway.tailscale.mode serve
```

### Docker 部署

```bash
# 拉取镜像
docker pull ghcr.io/openclaw/openclaw:latest

# 运行
docker run -d \
  -v ~/.openclaw:~/.openclaw \
  -p 18789:18789 \
  -e OPENCLAW_MODELS_ANTHROPIC_API_KEY=sk-ant-xxx \
  ghcr.io/openclaw/openclaw
```

---

## 使用指南

### CLI 命令参考

#### 网关管理

```bash
# 启动网关
openclaw gateway run

# 停止网关
openclaw gateway stop

# 查看状态
openclaw gateway status

# 重启网关
openclaw gateway restart

# 查看日志
openclaw logs
```

#### 消息发送

```bash
# 发送文本消息
openclaw message send --to +1234567890 --message "Hello"

# 发送图片
openclaw message send --to +1234567890 --file image.jpg

# 群发消息
openclaw message broadcast --channel telegram --message "Hello all"
```

#### Agent 交互

```bash
# 发送消息给 Agent
openclaw agent --message "总结今天的工作"

# 指定思考级别
openclaw agent --message "分析这个问题" --thinking high

# 开启详细模式
openclaw agent --message "解释这个代码" --verbose on
```

#### 渠道管理

```bash
# 列出渠道
openclaw channels list

# 查看渠道状态
openclaw channels status

# 渠道健康检查
openclaw channels status --probe

# 解决渠道问题
openclaw channels resolve telegram
```

#### 配对管理

```bash
# 查看待批准请求
openclaw pairing pending

# 批准配对
openclaw pairing approve telegram 123456

# 列出已配对设备
openclaw pairing list

# 撤销配对
openclaw pairing revoke device-uuid
```

#### 配置管理

```bash
# 查看配置
openclaw config get

# 设置配置项
openclaw config set gateway.port 18790

# 删除配置项
openclaw config unset gateway.auth.token

# 验证配置
openclaw config validate
```

### 聊天命令

在支持的消息渠道内发送以下命令：

| 命令 | 功能 |
|------|------|
| `/status` | 查看会话状态 |
| `/new` 或 `/reset` | 重置会话 |
| `/compact` | 压缩会话上下文 |
| `/think <level>` | 设置思考级别 |
| `/verbose on\|off` | 切换详细模式 |
| `/usage off\|tokens\|full` | 使用量显示 |
| `/restart` | 重启网关（群主专用） |
| `/activation mention\|always` | 群组激活模式 |

### Web 控制台

访问 `http://localhost:18789` 打开 Web 控制台：

- 查看网关状态
- 管理渠道
- 查看日志
- 配置设置

### macOS 应用

macOS 应用提供：

- 菜单栏控制
- Voice Wake 唤醒词
- Talk Mode 语音模式
- WebChat 集成

**安装**：
```bash
# 构建应用
pnpm mac:package

# 或使用预构建版本
# 从 GitHub Releases 下载
```

---

## 插件开发

### 创建插件

#### 1. 基本结构

```
my-plugin/
├── package.json
├── openclaw.plugin.json
└── src/
    └── index.ts
```

#### 2. package.json

```json
{
  "name": "@openclaw/my-plugin",
  "version": "1.0.0",
  "type": "module",
  "openclaw": {
    "extensions": ["./src/index.ts"]
  },
  "dependencies": {
    "openclaw": "latest"
  }
}
```

#### 3. openclaw.plugin.json

```json
{
  "id": "my-plugin",
  "name": "My Plugin",
  "version": "1.0.0",
  "description": "插件描述",
  "configSchema": {
    "type": "object",
    "properties": {
      "apiKey": {
        "type": "string"
      }
    }
  },
  "uiHints": {
    "apiKey": {
      "label": "API 密钥",
      "sensitive": true
    }
  }
}
```

#### 4. 插件入口

```typescript
export default function register(api) {
  // 注册工具
  api.registerTool({
    name: 'my_tool',
    description: '工具描述',
    inputSchema: {
      type: 'object',
      properties: {
        query: { type: 'string' }
      }
    },
    async execute(params, ctx) {
      // 工具逻辑
      return { result: 'ok' };
    }
  });

  // 注册 HTTP 路由
  api.registerHttpRoute({
    path: '/my-plugin/webhook',
    auth: 'plugin',
    handler: async (req, res) => {
      res.statusCode = 200;
      res.end('ok');
      return true;
    }
  });

  // 注册 CLI 命令
  api.registerCli(({ program }) => {
    program
      .command('mycmd')
      .description('命令描述')
      .action(() => {
        console.log('Hello');
      });
  });

  // 注册自动回复命令
  api.registerCommand({
    name: 'mystatus',
    description: '显示状态',
    handler: (ctx) => ({
      text: `插件运行中！渠道：${ctx.channel}`
    })
  });
}
```

### 渠道插件开发

```typescript
import type { ChannelPlugin } from 'openclaw/plugin-sdk/core';

const myChannel: ChannelPlugin = {
  id: 'mychannel',
  meta: {
    id: 'mychannel',
    label: 'MyChannel',
    selectionLabel: 'MyChannel (API)',
    docsPath: '/channels/mychannel',
    blurb: '我的渠道插件',
    aliases: ['mc']
  },
  capabilities: {
    chatTypes: ['direct', 'group'],
    mediaTypes: ['image', 'video']
  },
  config: {
    listAccountIds: (cfg) =>
      Object.keys(cfg.channels?.mychannel?.accounts ?? {}),
    resolveAccount: (cfg, accountId) =>
      cfg.channels?.mychannel?.accounts?.[accountId ?? 'default']
  },
  outbound: {
    deliveryMode: 'direct',
    sendText: async ({ text, to }) => {
      // 发送消息逻辑
      return { ok: true, messageId: '123' };
    }
  }
};

export default function(api) {
  api.registerChannel({ plugin: myChannel });
}
```

### 测试插件

```typescript
import { describe, it, expect } from 'vitest';
import { createTestPluginAPI } from 'openclaw/plugin-sdk/test-utils';
import myPlugin from './src/index';

describe('MyPlugin', () => {
  it('should register tool', async () => {
    const api = createTestPluginAPI();
    myPlugin(api);

    const tools = api.registeredTools;
    expect(tools).toContainEqual(
      expect.objectContaining({ name: 'my_tool' })
    );
  });
});
```

### 发布插件

```bash
# 构建
pnpm build

# 测试
pnpm test

# 发布到 npm
npm publish --access public

# 用户安装
openclaw plugins install @openclaw/my-plugin
```

---

## 故障排查

### 常见问题

#### 1. 网关无法启动

**症状**：
```
Error: Port 18789 is in use
```

**解决**：
```bash
# 查找占用端口的进程
lsof -i :18789

# 或使用不同端口
openclaw gateway run --port 18790
```

#### 2. 渠道连接失败

**症状**：
```
Failed to connect to Telegram
```

**解决**：
```bash
# 检查令牌
openclaw config get channels.telegram

# 重新认证
openclaw channels configure telegram

# 查看日志
openclaw logs --channel telegram
```

#### 3. 模型认证失败

**症状**：
```
Invalid API key for Anthropic
```

**解决**：
```bash
# 重新登录
openclaw models auth login --provider anthropic

# 检查环境变量
echo $ANTHROPIC_API_KEY

# 验证认证
openclaw models auth status --provider anthropic
```

#### 4. 配对失败

**症状**：
```
Device pairing failed
```

**解决**：
```bash
# 查看待批准请求
openclaw pairing pending

# 检查设备令牌
openclaw config get gateway.auth

# 重置配对
openclaw pairing revoke <device-id>
```

### 诊断工具

```bash
# 运行诊断
openclaw doctor

# 检查配置
openclaw config validate

# 渠道健康检查
openclaw channels status --probe

# 查看日志
openclaw logs --follow

# 导出诊断报告
openclaw doctor --export report.json
```

### 日志级别

```bash
# 设置日志级别
openclaw config set logging.level debug

# 查看特定模块日志
openclaw logs --module gateway

# 导出日志
openclaw logs --export logs.txt
```

---

## 安全最佳实践

### 1. 认证安全

```json5
{
  gateway: {
    auth: {
      mode: 'password',          // 使用密码认证
      allowTailscale: false,     // 不信任 Tailscale 身份
      tokenRotationDays: 30      // 定期轮换令牌
    }
  }
}
```

### 2. 渠道安全

```json5
{
  channels: {
    telegram: {
      dmPolicy: 'pairing',       // DM 需要配对
      allowFrom: ['admin-id']    // 白名单
    }
  }
}
```

### 3. 工具安全

```json5
{
  tools: {
    bash: {
      approval: 'manual',        // 手动批准
      denyPatterns: [
        'rm -rf',
        'sudo',
        'curl.*\\|.*sh'
      ]
    }
  }
}
```

### 4. 插件安全

```json5
{
  plugins: {
    allow: ['trusted-plugin-1', 'trusted-plugin-2'],
    entries: {
      'voice-call': {
        hooks: {
          allowPromptInjection: false  // 禁止提示注入
        }
      }
    }
  }
}
```

---

## 附录

### A. 环境变量

| 变量 | 描述 | 默认值 |
|------|------|--------|
| `OPENCLAW_CONFIG_PATH` | 配置文件路径 | `~/.openclaw/config.json` |
| `OPENCLAW_LOG_LEVEL` | 日志级别 | `info` |
| `OPENCLAW_GATEWAY_TOKEN` | 网关令牌 | - |
| `ANTHROPIC_API_KEY` | Anthropic API 密钥 | - |
| `OPENAI_API_KEY` | OpenAI API 密钥 | - |

### B. 文件结构

```
~/.openclaw/
├── config.json          # 主配置
├── sessions/            # 会话存储
│   └── <session-id>.db
├── credentials/         # 凭据存储
│   └── <provider>.json
├── extensions/          # 安装的插件
│   └── <plugin-id>/
└── logs/               # 日志文件
```

### C. 资源链接

- 官方网站：https://openclaw.ai
- 文档：https://docs.openclaw.ai
- GitHub：https://github.com/openclaw/openclaw
- Discord：https://discord.gg/clawd
- 愿景文档：VISION.md
- 贡献指南：CONTRIBUTING.md

---

*文档最后更新：2026-03-10*
*OpenClaw 版本：2026.3.9*
