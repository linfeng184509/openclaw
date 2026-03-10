# 第 5 章：消息通道配置

> 本章概述：详细讲解如何配置 OpenClaw 与各种消息平台（WhatsApp、Telegram、Discord、Slack 等）的连接。包括访问控制、群组管理、消息路由等核心主题。

## 学习目标

- 掌握主流消息通道的配置方法
- 理解 DM 配对和访问控制策略
- 学会配置群组消息和提及规则
- 了解消息路由和会话隔离机制

## 前置条件

- 已完成 Gateway 基础配置
- 拥有要连接的消息平台账号

---

## 5.1 支持的通道概览

OpenClaw 支持 20+ 消息平台，每个通道都有独立的配置和访问控制。

### 5.1.1 通道分类

| 类型 | 平台 | 状态 |
|------|------|------|
| **即时通讯** | WhatsApp | 生产就绪 |
| **即时通讯** | Telegram | 生产就绪 |
| **即时通讯** | Signal | 生产就绪 |
| **团队协作** | Slack | 生产就绪 |
| **团队协作** | Discord | 生产就绪 |
| **团队协作** | Microsoft Teams | 插件 |
| **苹果生态** | iMessage (BlueBubbles) | 推荐 |
| **苹果生态** | iMessage (legacy) | 已弃用 |
| **开源协议** | Matrix, Nostr, IRC | 插件 |
| **其他** | Google Chat, LINE, Feishu | 生产就绪 |

<Note>
**通道独立性**：
- 可以同时配置多个通道
- 每个通道有独立的访问控制
- 消息路由根据通道类型自动隔离
</Note>

### 5.1.2 通道配置结构

```json5
{
  channels: {
    // 通道通用配置
    defaults: {
      groupPolicy: "allowlist",  // 默认群组策略
    },

    // 具体通道配置
    whatsapp: { /* WhatsApp 配置 */ },
    telegram: { /* Telegram 配置 */ },
    discord: { /* Discord 配置 */ },
    slack: { /* Slack 配置 */ }
  }
}
```

---

## 5.2 WhatsApp 通道配置

### 5.2.1 快速设置

**步骤 1：配置访问策略**
```json5
{
  channels: {
    whatsapp: {
      dmPolicy: "pairing",           // DM 配对策略
      allowFrom: ["+15551234567"],   // 允许的用户
      groupPolicy: "allowlist",      // 群组策略
      groupAllowFrom: ["+15551234567"]
    }
  }
}
```

**步骤 2：链接 WhatsApp 设备（扫码）**
```bash
# 链接默认账户
openclaw channels login --channel whatsapp

# 链接特定账户
openclaw channels login --channel whatsapp --account work
```

**步骤 3：启动 Gateway**
```bash
openclaw gateway
```

**步骤 4：批准配对请求**
```bash
# 查看待批准的配对
openclaw pairing list whatsapp

# 批准配对
openclaw pairing approve whatsapp <CODE>
```

<Note>
**配对码有效期**：1 小时，超时需重新发起配对
</Note>

### 5.2.2 DM 访问控制

`dmPolicy` 控制直接消息的访问策略：

| 策略 | 行为 | 适用场景 |
|------|------|----------|
| `pairing` | 未知发送者收到配对码（默认） | 大多数场景 |
| `allowlist` | 仅允许列表中的用户 | 私人助理 |
| `open` | 允许所有 DM（需 `allowFrom: ["*"]`） | 公共机器人 |
| `disabled` | 拒绝所有 DM | 仅群组使用 |

**配置示例**：
```json5
{
  channels: {
    whatsapp: {
      // 允许列表模式
      dmPolicy: "allowlist",
      allowFrom: ["+15551234567", "+15559876543"],

      // 或使用开放模式
      // dmPolicy: "open",
      // allowFrom: ["*"]
    }
  }
}
```

### 5.2.3 群组配置

群组访问有两层控制：

**1. 群组成员资格（groups 配置）**
```json5
{
  channels: {
    whatsapp: {
      // 如果不设置 groups，所有群组都合格
      // 如果设置 groups，则作为允许列表
      groups: ["*"],  // "*" 允许所有群组

      // 或指定特定群组
      // groups: {
      //   "1234567890@g.us": { requireMention: true }
      // }
    }
  }
}
```

**2. 群组发送者策略（groupPolicy）**
```json5
{
  channels: {
    whatsapp: {
      groupPolicy: "allowlist",  // open | allowlist | disabled

      // 发送者允许列表
      groupAllowFrom: ["+15551234567"],

      // 提及要求
      groups: {
        "*": {
          requireMention: true  // 需要 @ 提及才能响应
        }
      }
    }
  }
}
```

### 5.2.4 个人号码 vs 专用号码

**推荐：专用号码模式**
- 独立的 WhatsApp 身份
- 更清晰的 DM 允许列表
- 降低自聊混淆风险

```json5
{
  channels: {
    whatsapp: {
      dmPolicy: "allowlist",
      allowFrom: ["+15551234567"],  // 仅允许特定用户
    }
  }
}
```

**个人号码模式**（需要自聊天保护）：
```json5
{
  channels: {
    whatsapp: {
      dmPolicy: "allowlist",
      allowFrom: ["+15551234567"],  // 你的号码
      selfChatMode: true            // 启用自聊天保护
    }
  }
}
```

### 5.2.5 消息分块和媒体

**文本分块配置**：
```json5
{
  channels: {
    whatsapp: {
      textChunkLimit: 4000,      // 每块最大字符数
      chunkMode: "length"        // "length" | "newline"
    }
  }
}
```

**媒体配置**：
```json5
{
  channels: {
    whatsapp: {
      mediaMaxMb: 50,            // 媒体大小限制（MB）
      sendReadReceipts: true,    // 发送已读回执
      ackReaction: {
        emoji: "👀",             // 确认反应 emoji
        direct: true,            // 直接消息启用
        group: "mentions"        // always | mentions | never
      }
    }
  }
}
```

### 5.2.6 故障排除

**问题 1：未链接（需要二维码）**
```bash
openclaw channels login --channel whatsapp
openclaw channels status
```

**问题 2：已链接但断开连接**
```bash
openclaw doctor
openclaw logs --follow
```

**问题 3：群组消息被忽略**
检查顺序：
1. `groupPolicy` 配置
2. `groupAllowFrom` 列表
3. `groups` 允许列表
4. 提及门控设置

---

## 5.3 Telegram 通道配置

### 5.3.1 快速设置

**步骤 1：创建 Bot Token**
1. 在 Telegram 中与 **@BotFather** 对话
2. 发送 `/newbot`，按提示操作
3. 保存返回的 token

**步骤 2：配置 Token 和 DM 策略**
```json5
{
  channels: {
    telegram: {
      enabled: true,
      botToken: "123:ABCDEF",    // BotFather 提供的 token
      dmPolicy: "pairing",
      groups: {
        "*": {
          requireMention: true
        }
      }
    }
  }
}
```

**步骤 3：启动 Gateway 并批准配对**
```bash
openclaw gateway
openclaw pairing list telegram
openclaw pairing approve telegram <CODE>
```

**步骤 4：将 Bot 添加到群组**
将 Bot 添加到群组后，配置 `groups` 和 `groupPolicy`。

<Note>
**Token 解析顺序**：
- 配置值优先于环境变量
- `TELEGRAM_BOT_TOKEN` 仅适用于默认账户
</Note>

### 5.3.2 查找你的 Telegram 用户 ID

**方法 1：查看日志（无需第三方 Bot）**
```bash
# 1. 给你的 Bot 发送 DM
# 2. 运行日志查看
openclaw logs --follow
# 3. 查找 from.id 字段
```

**方法 2：使用 Bot API**
```bash
curl "https://api.telegram.org/bot<BOT_TOKEN>/getUpdates"
```

**方法 3：第三方 Bot**
- `@userinfobot`
- `@getidsbot`

### 5.3.3 隐私模式和群组可见性

Telegram Bot 默认启用**隐私模式**，限制接收群组消息。

**选项 1：禁用隐私模式**
```
在 BotFather 中：/setprivacy -> Disable
```

**选项 2：将 Bot 设为管理员**
在群组设置中将 Bot 设为管理员。

**注意**：切换隐私模式后，需要移除并重新添加 Bot 到群组。

### 5.3.4 群组配置

**允许任何成员在特定群组**：
```json5
{
  channels: {
    telegram: {
      groups: {
        "-1001234567890": {
          groupPolicy: "open",     // 允许所有成员
          requireMention: false    // 不需要提及
        }
      }
    }
  }
}
```

**使用通配符**：
```json5
{
  channels: {
    telegram: {
      groups: {
        "*": {
          requireMention: true,   // 所有群组需要提及
          allowFrom: ["123456789"] // 特定用户允许列表
        }
      }
    }
  }
}
```

### 5.3.5 论坛话题和话题隔离

Telegram 论坛话题支持话题级隔离：

```json5
{
  channels: {
    telegram: {
      groups: {
        "-1001234567890": {
          topics: {
            "1": { agentId: "main" },      // 常规话题 → 主代理
            "3": { agentId: "dev" },       // 开发话题 → 开发代理
            "5": { agentId: "coder" }      // 代码审查 → 编程代理
          }
        }
      }
    }
  }
}
```

每个话题有独立的会话键：
```
agent:dev:telegram:group:-1001234567890:topic:3
```

### 5.3.6 长轮询 vs Webhook

**默认：长轮询**
无需额外配置，Grammy 自动处理。

**Webhook 模式**：
```json5
{
  channels: {
    telegram: {
      webhookUrl: "https://your-domain.com/telegram-webhook",
      webhookSecret: "your-secret-token",
      webhookPath: "/telegram-webhook",
      webhookHost: "127.0.0.1",
      webhookPort: 8787
    }
  }
}
```

---

## 5.4 Discord 通道配置

### 5.4.1 创建 Discord 应用和 Bot

**步骤 1：创建应用**
1. 访问 [Discord Developer Portal](https://discord.com/developers/applications)
2. 点击 **New Application**
3. 命名应用（如 "OpenClaw"）

**步骤 2：创建 Bot**
1. 点击左侧 **Bot**
2. 点击 **Add Bot**
3. 设置用户名

**步骤 3：启用特权意图**
在 **Privileged Gateway Intents** 中启用：
- **Message Content Intent**（必需）
- **Server Members Intent**（推荐）
- **Presence Intent**（可选）

**步骤 4：复制 Bot Token**
点击 **Reset Token**，复制并保存。

**步骤 5：生成邀请 URL**
1. 点击 **OAuth2** → **URL Generator**
2. 选择 scopes: `bot`, `applications.commands`
3. 选择权限：
   - View Channels
   - Send Messages
   - Read Message History
   - Embed Links
   - Attach Files
   - Add Reactions（可选）

**步骤 6：启用开发者模式并收集 ID**
1. Discord 设置 → 高级 → 启用开发者模式
2. 右键服务器图标 → 复制服务器 ID
3. 右键用户头像 → 复制用户 ID

### 5.4.2 配置 Discord

**步骤 0：设置 Bot Token**
```bash
openclaw config set channels.discord.token '"YOUR_BOT_TOKEN"' --json
openclaw config set channels.discord.enabled true --json
```

**步骤 1：配置访问策略**
```json5
{
  channels: {
    discord: {
      enabled: true,
      token: "YOUR_BOT_TOKEN",
      groupPolicy: "allowlist",
      guilds: {
        "123456789012345678": {
          requireMention: true,
          users: ["987654321098765432"],
          roles: ["123456789012345678"]
        }
      }
    }
  }
}
```

**步骤 2：批准 DM 配对**
给 Bot 发送 DM，收到配对码后：
```bash
openclaw pairing approve discord <CODE>
```

### 5.4.3 服务器工作区配置

**添加服务器到允许列表**：
```json5
{
  channels: {
    discord: {
      groupPolicy: "allowlist",
      guilds: {
        "YOUR_SERVER_ID": {
          requireMention: false,  // 私人服务器可设为 false
          users: ["YOUR_USER_ID"]
        }
      }
    }
  }
}
```

### 5.4.4 提及门控

群组消息默认需要提及才能响应：

**配置提及模式**：
```json5
{
  agents: {
    list: [{
      id: "main",
      groupChat: {
        mentionPatterns: ["@openclaw", "claw"]  // 提及模式
      }
    }]
  }
}
```

**会话级命令**：
- `/activation mention` — 仅提及时响应
- `/activation always` — 总是响应

### 5.4.5 原生命令和 Slash Commands

Discord 支持原生 Slash Commands：

```json5
{
  commands: {
    native: "auto"  // 自动启用 Discord 原生命令
  },
  channels: {
    discord: {
      commands: {
        native: true
      }
    }
  }
}
```

**可用命令**：
- `/model` — 切换模型
- `/status` — 查看状态
- `/new` — 新建会话
- `/compact` — 压缩上下文

---

## 5.5 Slack 通道配置

### 5.5.1 创建 Slack 应用

**步骤 1：创建应用**
1. 访问 [Slack API](https://api.slack.com/apps)
2. 点击 **Create New App** → **From scratch**
3. 命名应用并选择工作区

**步骤 2：配置 Bot Token**
1. 点击 **OAuth & Permissions**
2. 添加 Bot Scopes：
   - `chat:write`
   - `channels:read`
   - `groups:read`
   - `im:read`

**步骤 3：安装应用**
点击 **Install to Workspace**，复制 Bot Token。

### 5.5.2 配置 Slack

```json5
{
  channels: {
    slack: {
      enabled: true,
      botToken: "xoxb-your-bot-token",
      appToken: "xapp-your-app-token",  // 如果使用 Socket Mode
      dmPolicy: "pairing",
      groupPolicy: "allowlist"
    }
  }
}
```

---

## 5.6 通道通用配置

### 5.6.1 DM 配对机制

所有通道共享相似的 DM 配对流程：

**配对状态**：
```bash
# 查看待批准配对
openclaw pairing list [channel]

# 批准配对
openclaw pairing approve <channel> <CODE>

# 拒绝配对
openclaw pairing reject <channel> <CODE>

# 查看已批准设备
openclaw pairing approved --channel <channel>
```

### 5.6.2 消息路由规则

消息根据以下规则路由到会话：

```mermaid
flowchart TD
    A[消息到达] --> B{消息类型?}
    B -->|DM| C{dmScope 配置}
    B -->|群组| D[群组会话]
    B -->|话题| E[话题会话]

    C -->|main| F[agent:&lt;agentId&gt;:main]
    C -->|per-peer| G[agent:&lt;agentId&gt;:dm:&lt;peerId&gt;]
    C -->|per-channel-peer| H[agent:&lt;agentId&gt;:&lt;channel&gt;:dm:&lt;peerId&gt;]
    C -->|per-account-channel-peer| I[agent:&lt;agentId&gt;:&lt;account&gt;:&lt;channel&gt;:dm:&lt;peerId&gt;]

    D --> J[agent:&lt;agentId&gt;:&lt;channel&gt;:group:&lt;groupId&gt;]
    E --> K[agent:&lt;agentId&gt;:&lt;channel&gt;:group:&lt;groupId&gt;:topic:&lt;threadId&gt;]
```

```
DM 消息：
- dmScope: "main"           → agent:<agentId>:main
- dmScope: "per-peer"       → agent:<agentId>:dm:<peerId>
- dmScope: "per-channel-peer" → agent:<agentId>:<channel>:dm:<peerId>

群组消息：
- 默认 → agent:<agentId>:<channel>:group:<groupId>

话题消息（Telegram/Discord）：
- 默认 → agent:<agentId>:<channel>:group:<groupId>:topic:<threadId>
```

### 5.6.3 通道状态检查

```bash
# 查看所有通道状态
openclaw channels status

# 探测特定通道
openclaw channels status --probe --channel discord

# 健康检查
openclaw health
```

---

## 本章小结

- **WhatsApp**：扫码配对，支持个人号码和专用号码模式
- **Telegram**：Bot Token 认证，支持论坛话题隔离
- **Discord**：需要创建应用，支持 Slash Commands
- **Slack**：Bot Token + App Token（Socket Mode）
- **通用配置**：DM 配对、群组策略、提及门控

## 延伸阅读

- [通道故障排除](https://docs.openclaw.ai/channels/troubleshooting)
- [配对机制详解](https://docs.openclaw.ai/channels/pairing)
- [群组消息配置](https://docs.openclaw.ai/channels/groups)
- [第 6 章：模型提供商配置](chapter-06.md)

---

*上一章：[第 4 章：网关配置详解](chapter-04.md) | 下一章：[第 6 章：模型提供商配置](chapter-06.md)*
