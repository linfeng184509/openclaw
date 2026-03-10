# OpenClaw 系统运行机制与协作图

> 通过序列图和流程图深入解析系统运行机制
> 版本：2026.3.9
> 最后更新：2026-03-10

---

## 目录

1. [系统启动流程](#1-系统启动流程)
2. [消息接收与处理](#2-消息接收与处理)
3. [Agent 响应生成](#3-agent-响应生成)
4. [消息发送流程](#4-消息发送流程)
5. [设备配对流程](#5-设备配对流程)
6. [工具执行流程](#6-工具执行流程)
7. [会话管理流程](#7-会话管理流程)
8. [插件加载流程](#8-插件加载流程)

---

## 1. 系统启动流程

### 1.1 Gateway 启动序列图

```mermaid
sequenceDiagram
    participant User as 用户
    participant CLI as CLI 入口
    participant Config as 配置系统
    participant Gateway as Gateway 核心
    participant Channels as 渠道管理器
    participant Plugins as 插件运行时
    participant WebSocket as WebSocket 服务器
    participant DB as SQLite 存储

    User->>CLI: openclaw gateway run
    CLI->>Config: loadConfig()
    Config->>Config: 合并多层配置
    Config-->>CLI: 返回 Config 对象

    CLI->>Gateway: new Gateway(config)
    Gateway->>DB: 初始化会话存储
    DB-->>Gateway: 会话存储就绪

    Gateway->>WebSocket: 创建服务器
    WebSocket->>WebSocket: bind(127.0.0.1:18789)
    WebSocket-->>Gateway: 监听成功

    Gateway->>Plugins: discoverPlugins()
    Plugins->>Plugins: 扫描 4 个目录
    Plugins-->>Gateway: 返回插件列表

    Gateway->>Channels: loadChannels()
    loop 每个配置的渠道
        Channels->>Channels: 初始化渠道适配器
        Channels->>Channels: 连接渠道 API
        Channels-->>Gateway: 渠道就绪
    end

    Gateway->>Gateway: 启动后台服务
    Gateway-->>CLI: Gateway 启动完成
    CLI-->>User: 显示启动日志
```

### 1.2 启动流程详细说明

```typescript
// 1. 配置加载 (src/config/io.ts)
async function loadConfig(): Promise<Config> {
  const sources = [
    { path: 'defaults', data: DEFAULT_CONFIG, priority: 0 },
    { path: '~/.openclaw/config.json', priority: 1 },
    { path: './.openclaw/config.json', priority: 2 },
    { path: 'env', data: parseEnvConfig(process.env), priority: 3 },
  ];

  // 合并配置 (高优先级覆盖低优先级)
  const merged = mergeConfigs(sources);

  // 验证配置 Schema
  await validateConfigObjectWithPlugins(merged);

  return merged as Config;
}

// 2. Gateway 初始化 (src/gateway/gateway.ts)
class Gateway {
  async start(): Promise<void> {
    // 2.1 初始化会话存储
    this.sessions = await createSessionStore(this.config);

    // 2.2 启动 WebSocket 服务器
    await this.startWebSocketServer({
      bind: this.config.gateway.bind,
      port: this.config.gateway.port,
    });

    // 2.3 发现并加载插件
    const { plugins, errors } = await discoverPlugins(this.config);
    for (const plugin of plugins) {
      await loadPlugin(plugin, this.createPluginAPI());
    }

    // 2.4 初始化渠道
    for (const [channelId, channelConfig] of Object.entries(this.config.channels)) {
      await this.channels.load(channelId, channelConfig);
    }

    // 2.5 启动后台服务
    await this.startHealthMonitor();
    await this.startCronScheduler();
  }
}
```

### 1.3 渠道初始化流程图

```mermaid
flowchart TD
    A[开始加载渠道] --> B{读取渠道配置}
    B -->|有配置 | C[创建渠道实例]
    B -->|无配置 | D[跳过该渠道]

    C --> E{渠道类型?}
    E -->|Telegram| F[初始化 grammY Bot]
    E -->|Discord| G[初始化 discord.js Client]
    E -->|Slack| H[初始化 Slack Bolt App]
    E -->|WhatsApp| I[初始化 Baileys Socket]
    E -->|Signal| J[启动 signal-cli 进程]

    F --> K[连接 Telegram API]
    G --> K[连接 Discord Gateway]
    H --> K[连接 Slack RTM/WebSocket]
    I --> K[连接 WhatsApp WebSocket]
    J --> K[等待 signal-cli 就绪]

    K --> L{连接成功？}
    L -->|是 | M[注册消息处理器]
    L -->|否 | N[记录错误并重试]

    M --> O[渠道就绪]
    N --> K
```

---

## 2. 消息接收与处理

### 2.1 消息接收序列图 (以 Telegram 为例)

```mermaid
sequenceDiagram
    participant TG as Telegram API
    participant Monitor as 渠道 Monitor
    participant Router as 消息路由器
    participant Session as 会话管理器
    participant Agent as Agent 运行时
    participant Channel as 渠道发送器

    TG->>Monitor: 推送新消息 (Update)
    Monitor->>Monitor: 解析消息体

    Note over Monitor: bot-message-context.ts<br/>提取会话上下文

    Monitor->>Monitor: 标准化消息格式
    Monitor->>Router: routeMessage(normalizedMsg)

    Router->>Router: 提取关键信息
    Note over Router: - channelType: telegram<br/>- accountId: bot-id<br/>- peerId: user-id<br/>- threadId: message-id

    Router->>Session: deriveSessionKey(channel, account, peer)
    Session-->>Router: session-key-abc123

    Router->>Session: loadSession(sessionKey)
    Session->>Session: 从 SQLite 加载
    Session-->>Router: 返回 SessionEntry

    Router->>Agent: enqueueMessage({
  session,
  message: text,
  sender
})

    Note over Agent: 消息进入队列<br/>等待 Agent 处理

    Router-->>Monitor: 路由完成
    Monitor-->>TG: ack
```

### 2.2 消息标准化流程

```typescript
// src/telegram/bot-message-context.ts

interface NormalizedMessage {
  id: string;           // Telegram message_id
  channelId: string;    // 'telegram'
  accountId: string;    // Bot ID
  peerId: string;       // User/Chat ID
  threadId?: string;    // 回复线程 ID
  sender: {
    id: string;
    name: string;
    isBot: boolean;
  };
  content: {
    type: 'text' | 'photo' | 'voice' | 'video';
    text?: string;
    mediaUrl?: string;
  };
  timestamp: number;
  isIncoming: boolean;
}

function normalizeTelegramMessage(update: Update): NormalizedMessage {
  const message = update.message || update.edited_message;

  return {
    id: String(message.message_id),
    channelId: 'telegram',
    accountId: String(message.bot.id),
    peerId: String(message.chat.id),
    threadId: message.is_topic_message
      ? String(message.message_thread_id)
      : undefined,
    sender: {
      id: String(message.from.id),
      name: message.from.first_name,
      isBot: message.from.is_bot || false,
    },
    content: {
      type: message.photo ? 'photo' :
            message.voice ? 'voice' : 'text',
      text: message.text || message.caption,
      mediaUrl: message.photo
        ? message.photo[message.photo.length - 1].file_id
        : undefined,
    },
    timestamp: message.date * 1000,
    isIncoming: true,
  };
}
```

### 2.3 消息路由决策树

```mermaid
flowchart TD
    A[收到消息] --> B{是机器人自己？}
    B -->|是 | C[忽略消息]
    B -->|否 | D{是群组消息？}

    D -->|是 | E{提到机器人？}
    E -->|否 | F{群组激活模式？}
    E -->|是 | G[处理消息]
    F -->|mention | H[忽略]
    F -->|always | G

    D -->|否 | I{是 DM 消息？}
    I -->|是 | J{发送者在白名单？}
    J -->|是 | G
    J -->|否 | K{DM 策略？}

    K -->|pairing | L[发送配对码]
    K -->|open | G

    I -->|否 | M[未知消息类型，忽略]

    G --> N[派发到 Agent 队列]
    L --> O[等待用户配对]
```

---

## 3. Agent 响应生成

### 3.1 Agent 循环序列图

```mermaid
sequenceDiagram
    participant Queue as 消息队列
    participant Loop as Agent Loop
    participant Session as 会话存储
    participant Hooks as 钩子系统
    participant Context as 上下文引擎
    participant Model as 模型提供者
    participant Tools as 工具执行器
    participant Stream as 流式处理器

    Queue->>Loop: 出队消息
    Loop->>Session: 加载会话

    Note over Hooks: before_model_resolve 钩子
    Loop->>Hooks: runBeforeModelResolve(ctx)
    Hooks-->>Loop: modelOverride?

    Loop->>Loop: 解析模型配置

    Note over Context: 构建提示
    Loop->>Context: assemble({ session, message })
    Context->>Session: 读取历史消息
    Context->>Context: 应用 token 预算
    Context-->>Loop: 返回 Prompt

    Note over Hooks: before_prompt_build 钩子
    Loop->>Hooks: runBeforePromptBuild(ctx)
    Hooks-->>Loop: prepend/append 内容

    Loop->>Model: completeStream(prompt)

    par 流式处理
      Stream->>Stream: 接收文本块
      Stream-->>Client: 推送文本到渠道
    and 工具调用
      Model->>Tools: 工具调用请求
      Tools->>Tools: 执行工具
      Tools-->>Model: 返回结果
    end

    Model-->>Loop: 完成响应

    Loop->>Context: ingest({ message, response })
    Context->>Session: 更新历史记录

    Loop->>Session: 保存会话

    Note over Loop: 检查是否需要压缩
    Loop->>Loop: shouldCompact()?
    Loop->>Context: compact()
```

### 3.2 提示构建流程

```mermaid
flowchart LR
    A[用户消息] --> B[系统提示]

    B --> C[基础系统提示]
    B --> D[钩子 prepend]
    B --> E[钩子 append]

    C --> F[最终系统提示]
    D --> F
    E --> F

    F --> G[上下文消息]

    G --> H[历史对话 N 条]
    G --> I[ artifacts/文件]
    G --> J[工具定义]

    H --> K[完整 Prompt]
    I --> K
    J --> K
    F --> K

    K --> L[发送给模型]
```

### 3.3 流式响应处理

```typescript
// src/agents/agent-loop.ts

async function processStream(
  stream: AsyncIterable<StreamChunk>,
  handlers: {
    onChunk: (text: string) => void;
    onToolCall: (toolCall: ToolCall) => Promise<ToolResult>;
  }
): Promise<AgentResult> {
  const result: AgentResult = {
    text: '',
    toolCalls: [],
  };

  for await (const chunk of stream) {
    if (chunk.type === 'content') {
      // 文本内容
      result.text += chunk.text;
      handlers.onChunk(chunk.text);
    }

    if (chunk.type === 'tool_call') {
      // 工具调用
      const toolCall = chunk.toolCall;
      result.toolCalls.push(toolCall);

      // 执行工具
      const toolResult = await handlers.onToolCall(toolCall);

      // 将结果返回给模型
      yield { type: 'tool_result', result: toolResult };
    }

    if (chunk.type === 'done') {
      // 完成
      result.summary = chunk.summary;
      break;
    }
  }

  return result;
}
```

---

## 4. 消息发送流程

### 4.1 消息发送序列图

```mermaid
sequenceDiagram
    participant Client as 客户端 (CLI/Web/App)
    participant Gateway as Gateway
    participant Router as 路由管理器
    participant Channel as 渠道适配器
    participant External as 外部渠道 API

    Client->>Gateway: req:send {
  channel: 'telegram',
  to: '+1234567890',
  text: 'Hello'
}

    Gateway->>Gateway: 验证请求参数

    Gateway->>Router: resolveChannel('telegram')
    Router-->>Gateway: TelegramChannel

    Gateway->>Channel: sendText({
  to: '+1234567890',
  text: 'Hello',
  accountId: 'default'
})

    Channel->>Channel: 标准化消息格式

    alt 文本消息
        Channel->>External: Telegram Bot API
        External-->>Channel: { message_id: 123 }
    else 媒体消息
        Channel->>Channel: 上传媒体文件
        Channel->>External: sendPhoto/Video/Audio
        External-->>Channel: { file_id: '...' }
    end

    Channel-->>Gateway: { ok: true, messageId: '123' }

    Gateway->>Gateway: 更新会话 lastTo/lastChannel

    Gateway-->>Client: res:send { ok: true }
```

### 4.2 渠道发送适配器

```typescript
// src/telegram/bot/delivery.ts

interface SendTextParams {
  to: string;           // 接收者 ID
  text: string;         // 消息文本
  accountId: string;    // 发送账号
  threadId?: string;    // 回复线程
  replyTo?: string;     // 回复消息 ID
}

interface SendResult {
  ok: boolean;
  messageId: string;
  channel: string;
  timestamp: number;
}

class TelegramDelivery {
  async sendText(params: SendTextParams): Promise<SendResult> {
    const bot = this.getBot(params.accountId);

    // 处理长消息分片
    const chunks = this.chunkMessage(params.text);

    let lastMessageId: string | undefined;

    for (const chunk of chunks) {
      const result = await bot.api.sendMessage(params.to, chunk, {
        reply_to_message_id: params.replyTo,
        message_thread_id: params.threadId,
      });

      lastMessageId = String(result.message_id);

      // 第一条消息后清除 replyTo，后续消息不嵌套回复
      params.replyTo = undefined;
    }

    return {
      ok: true,
      messageId: lastMessageId!,
      channel: 'telegram',
      timestamp: Date.now(),
    };
  }

  private chunkMessage(text: string): string[] {
    const MAX_LENGTH = 4096;  // Telegram 限制

    if (text.length <= MAX_LENGTH) {
      return [text];
    }

    const chunks: string[] = [];
    let remaining = text;

    while (remaining.length > 0) {
      if (remaining.length <= MAX_LENGTH) {
        chunks.push(remaining);
        break;
      }

      // 在最后一个空格处截断
      let cutIndex = MAX_LENGTH;
      const lastSpace = remaining.lastIndexOf(' ', MAX_LENGTH);
      if (lastSpace > 0) {
        cutIndex = lastSpace;
      }

      chunks.push(remaining.slice(0, cutIndex));
      remaining = remaining.slice(cutIndex + 1);
    }

    return chunks;
  }
}
```

### 4.3 媒体消息发送流程

```mermaid
flowchart TD
    A[发送媒体消息] --> B{媒体类型？}

    B -->|图片 | C[准备图片]
    B -->|语音 | D[准备语音]
    B -->|视频 | E[准备视频]

    C --> F{是 URL 还是文件？}
    D --> F
    E --> F

    F -->|URL | G[直接发送 URL]
    F -->|文件 | H[上传到渠道服务器]

    H --> I{文件大小？}
    I -->|<=20MB | J[直接上传]
    I -->|>20MB | K[分块上传/错误]

    G --> L[调用渠道 API]
    J --> L
    K --> L

    L --> M{发送成功？}
    M -->|是 | N[返回 message_id]
    M -->|否 | O[重试或报错]

    O --> P{重试次数 < 3？}
    P -->|是 | L
    P -->|否 | Q[抛出错误]

    N --> R[更新会话记录]
```

---

## 5. 设备配对流程

### 5.1 完整配对序列图

```mermaid
sequenceDiagram
    participant NewDevice as 新设备
    participant Gateway as Gateway
    participant AdminChannel as 管理员渠道
    participant Admin as 管理员
    participant Store as 配对存储

    NewDevice->>Gateway: req:connect (首次)
    Gateway->>Gateway: 检查设备是否已配对
    Gateway-->>NewDevice: res:connect (需要配对)

    Gateway->>Gateway: 生成 6 位配对码
    Gateway->>Store: 保存配对请求 (5 分钟过期)

    Gateway->>AdminChannel: 发送配对通知
    Note over AdminChannel: "新设备请求配对:<br/>设备名：My Mac<br/>配对码：123456"

    Admin-->>AdminChannel: 看到配对通知

    Admin->>AdminChannel: /pairing approve 123456

    AdminChannel->>Gateway: 转发批准命令

    Gateway->>Gateway: 验证配对码
    Gateway->>Store: 查找配对请求
    Store-->>Gateway: 返回请求详情

    Gateway->>Gateway: 生成设备令牌
    Gateway->>Store: 保存配对设备信息

    Gateway->>AdminChannel: 发送批准通知
    AdminChannel-->>Admin: 显示批准结果

    Gateway-->>NewDevice: event:pairing_approved { deviceToken }

    NewDevice->>NewDevice: 存储设备令牌

    NewDevice->>Gateway: req:connect (带 deviceToken)
    Gateway->>Store: 验证设备令牌
    Store-->>Gateway: 验证通过
    Gateway-->>NewDevice: res:connect (成功)
```

### 5.2 配对码生成与验证

```typescript
// src/gateway/device-auth.ts

class DevicePairingManager {
  private pending = new Map<string, PairingRequest>();

  /**
   * 创建配对请求
   */
  async createPairingRequest(params: {
    deviceId: string;
    deviceInfo: DeviceInfo;
    channel: string;
  }): Promise<string> {
    // 生成 6 位数字配对码
    const code = crypto.randomInt(100000, 999999).toString();

    const request: PairingRequest = {
      code,
      deviceId: params.deviceId,
      deviceInfo: params.deviceInfo,
      channel: params.channel,
      requestedAt: Date.now(),
      expiresAt: Date.now() + 5 * 60 * 1000,  // 5 分钟过期
    };

    this.pending.set(code, request);

    // 发送配对通知到管理员渠道
    await this.sendPairingNotification(request);

    return code;
  }

  /**
   * 批准配对
   */
  async approvePairing(code: string): Promise<PairingResult> {
    const request = this.pending.get(code);
    if (!request) {
      return { ok: false, reason: 'code_not_found' };
    }

    // 检查是否过期
    if (Date.now() > request.expiresAt) {
      this.pending.delete(code);
      return { ok: false, reason: 'code_expired' };
    }

    // 生成设备令牌
    const deviceToken = await this.generateDeviceToken(request.deviceId);

    // 存储配对设备
    await this.pairedStore.set(request.deviceId, {
      deviceId: request.deviceId,
      deviceInfo: request.deviceInfo,
      deviceToken,
      pairedAt: Date.now(),
      pairedVia: request.channel,
    });

    // 清理配对请求
    this.pending.delete(code);

    // 发送批准通知
    await this.sendApprovalNotification(request);

    return { ok: true, deviceToken };
  }

  /**
   * 生成设备令牌 (使用 HMAC)
   */
  private async generateDeviceToken(deviceId: string): Promise<string> {
    const key = await crypto.subtle.generateKey(
      { name: 'HMAC', hash: 'SHA-256', length: 256 },
      true,
      ['sign', 'verify']
    );

    const exported = await crypto.subtle.exportKey('raw', key);
    const token = arrayBufferToBase64Url(exported);

    // 存储密钥用于后续验证
    this.deviceKeys.set(deviceId, key);

    return token;
  }
}
```

### 5.3 配对状态机

```mermaid
stateDiagram-v2
    [*] --> Unpaired: 新设备

    Unpaired --> PairingRequested: 发起连接
    PairingRequested --> PairingExpired: 5 分钟超时
    PairingRequested --> PairingApproved: 管理员批准
    PairingRequested --> PairingRejected: 管理员拒绝

    PairingApproved --> Paired: 存储令牌
    Paired --> Connected: 使用令牌连接

    Connected --> Disconnected: 断开连接
    Disconnected --> Connected: 重新连接

    Paired --> Revoked: 撤销配对
    Revoked --> Unpaired: 重新配对

    PairingExpired --> [*]: 清除请求
    PairingRejected --> [*]: 清除请求
    Revoked --> [*]: 清除设备
```

---

## 6. 工具执行流程

### 6.1 Bash 工具执行序列图

```mermaid
sequenceDiagram
    participant Model as AI 模型
    participant Agent as Agent Loop
    participant Approval as 批准管理器
    participant Admin as 管理员
    participant Executor as 执行器
    participant Process as 子进程

    Model->>Agent: 工具调用 {
  name: 'bash',
  input: { command: 'ls -la' }
}

    Agent->>Approval: shouldRequireApproval('ls -la')
    Approval->>Approval: 检查命令模式

    alt 命令在允许列表
        Approval-->>Agent: 不需要批准
    else 命令需要批准
        Approval-->>Agent: 需要批准

        Agent->>Admin: 发送批准请求
        Admin->>Admin: 审查命令

        alt 批准
            Admin->>Admin: /approve <request-id>
            Admin->>Agent: 批准通知
            Agent->>Approval: recordApproval()
        else 拒绝
            Admin->>Admin: /deny <request-id>
            Admin->>Agent: 拒绝通知
            Agent->>Model: 工具执行被拒绝
        end
    end

    Agent->>Executor: execute({ command, cwd, env })
    Executor->>Process: spawn(command, { shell: true })

    Process->>Executor: stdout 数据
    Executor-->>Agent: 流式输出
    Process->>Executor: stderr 数据
    Executor-->>Agent: 错误输出

    Process->>Executor: exit(code)
    Executor-->>Agent: { stdout, stderr, exitCode }

    Agent->>Model: 工具执行结果
```

### 6.2 命令风险评估

```typescript
// src/agents/bash-tools.exec-approval-request.ts

class ExecApprovalManager {
  private denyPatterns = [
    /rm\s+(-[rf]+\s+)?\//,           // 删除根目录
    /sudo\s+/,                        // 提权命令
    /curl.*\|\s*(ba)?sh/,             // 下载并执行
    /wget.*\|\s*(ba)?sh/,
    /chmod\s+[0-7]+777/,              // 设置可执行权限
    /mkfs/,                           // 格式化
    /dd\s+.*of=\/dev/,                // 直接写入设备
    /:\(\)\{\s*:\|:\s*&\}\s*;/,       // Shellshock
  ];

  private allowPatterns = [
    /^ls\s+/,                         // 列出文件
    /^cat\s+/,                        // 查看文件
    /^grep\s+/,                       // 搜索
    /^head\s+/,                       // 查看文件头
    /^tail\s+/,                       // 查看文件尾
    /^pwd$/,                          // 当前目录
    /^echo\s+/,                       // 输出文本
    /^date$/,                         // 日期
  ];

  async shouldRequireApproval(command: string): Promise<{
    requiresApproval: boolean;
    reason?: string;
  }> {
    // 检查拒绝模式 (总是需要批准)
    for (const pattern of this.denyPatterns) {
      if (pattern.test(command)) {
        return { requiresApproval: true, reason: 'denied_pattern' };
      }
    }

    // 检查允许模式 (自动批准)
    for (const pattern of this.allowPatterns) {
      if (pattern.test(command)) {
        return { requiresApproval: false };
      }
    }

    // 默认需要批准
    return { requiresApproval: true };
  }

  assessRisk(command: string): 'low' | 'medium' | 'high' {
    const highRiskPatterns = [
      /rm\s+(-[rf]+\s+)?\//,
      /sudo\s+/,
      /curl.*\|\s*(ba)?sh/,
    ];

    const mediumRiskPatterns = [
      /rm\s+/,                        // 删除文件
      /mv\s+.*\/dev/,                 // 移动到设备
      /echo\s+.*>\s*\//,              // 写入系统文件
    ];

    for (const pattern of highRiskPatterns) {
      if (pattern.test(command)) {
        return 'high';
      }
    }

    for (const pattern of mediumRiskPatterns) {
      if (pattern.test(command)) {
        return 'medium';
      }
    }

    return 'low';
  }
}
```

### 6.3 工具执行批准 UI (Telegram)

```mermaid
sequenceDiagram
    participant Agent as Agent
    participant TG as Telegram Bot
    participant Admin as 管理员
    participant Callback as 回调处理器

    Agent->>TG: 发送批准请求消息

    Note over TG: 消息内容:<br/>⚠️ 工具执行请求<br/>命令：ls -la /app<br/>风险：低<br/><br/>[✓ 批准] [✗ 拒绝]

    TG->>Admin: 显示内联按钮

    Admin->>TG: 点击"✓ 批准"
    TG->>Callback: 回调 { action: 'approve', requestId: '...' }

    Callback->>Callback: 验证回调来源
    Callback->>Agent: 转发批准决定

    Agent->>TG: 更新原消息
    Note over TG: 消息更新为:<br/>✅ 已批准 - 执行中...

    Agent->>Agent: 执行命令

    Agent->>TG: 更新结果
    Note over TG: ✅ 执行完成<br/>退出码：0<br/>输出：(前 1000 字符)
```

---

## 7. 会话管理流程

### 7.1 会话键派生流程

```mermaid
flowchart LR
    A[输入参数] --> B[规范化处理]

    B --> C[channelType<br/>toLowerCase]
    B --> D[accountId<br/>toLowerCase]
    B --> E[peerId<br/>trim]
    B --> F[groupId?<br/>optional]

    C --> G[组合键]
    D --> G
    E --> G
    F --> G

    G --> H[join '::']
    H --> I[SHA256 哈希]
    I --> J[截取 32 字符]
    J --> K[最终 sessionKey]
```

### 7.2 会话存储操作序列图

```mermaid
sequenceDiagram
    participant Client as 客户端
    participant Gateway as Gateway
    participant SessionStore as 会话存储
    participant Cache as LRU 缓存
    participant SQLite as SQLite 文件

    Client->>Gateway: req:sessions.list
    Gateway->>Cache: 检查缓存
    Cache-->>Gateway: 缓存未命中

    Gateway->>SQLite: 读取 session-store.json
    SQLite-->>Gateway: 返回 JSON 数据

    Gateway->>Gateway: 解析并规范化

    loop 每个会话条目
        Gateway->>Gateway: 规范化 deliveryContext
        Gateway->>Gateway: 合并遗留键
    end

    Gateway->>Cache: 写入缓存 (TTL 45s)
    Gateway-->>Client: 返回会话列表

    Note over Client,Gateway: 稍后...

    Client->>Gateway: req:sessions.patch
    Gateway->>Gateway: 获取写锁

    Gateway->>Cache: 更新缓存条目
    Gateway->>SQLite: 写入 session-store.json

    Gateway->>Gateway: 释放写锁
    Gateway-->>Client: res:patch { ok: true }
```

### 7.3 会话维护流程

```typescript
// src/config/sessions/store-maintenance.ts

interface MaintenanceConfig {
  maxEntries: number;           // 最大会话数：1000
  maxAgeDays: number;           // 最大保留天数：90
  pruneStaleAfterDays: number;  // 清理不活跃会话：30
}

async function maintainSessionStore(
  store: SessionStore,
  config: MaintenanceConfig
): Promise<MaintenanceResult> {
  const now = Date.now();
  const staleThreshold = now - config.pruneStaleAfterDays * 24 * 60 * 60 * 1000;
  const maxAgeThreshold = now - config.maxAgeDays * 24 * 60 * 60 * 1000;

  const toRemove: string[] = [];

  // 1. 清理过期会话
  for (const [key, entry] of Object.entries(store)) {
    if (entry.createdAt < maxAgeThreshold) {
      toRemove.push(key);
      continue;
    }
    if (entry.lastActivityAt < staleThreshold) {
      toRemove.push(key);
    }
  }

  // 2. 如果会话数超限，删除最旧的
  if (Object.keys(store).length > config.maxEntries) {
    const sorted = Object.entries(store)
      .sort((a, b) => a[1].lastActivityAt - b[1].lastActivityAt);

    const excessCount = Object.keys(store).length - config.maxEntries;
    for (let i = 0; i < excessCount; i++) {
      toRemove.push(sorted[i][0]);
    }
  }

  // 3. 执行删除
  for (const key of toRemove) {
    delete store[key];
  }

  return {
    removed: toRemove.length,
    remaining: Object.keys(store).length,
  };
}
```

### 7.4 会话压缩触发条件

```mermaid
flowchart TD
    A[新消息处理完成] --> B[检查压缩条件]

    B --> C{消息数量 > 阈值？}
    C -->|是 | D[触发压缩]
    C -->|否 | E{Token 数 > 预算？}

    E -->|是 | D
    E -->|否 | F{距离上次压缩 > 时间阈值？}

    F -->|是 | G{消息数量 > 最小压缩数？}
    F -->|否 | H[不压缩]

    G -->|是 | D
    G -->|否 | H

    D --> I[调用 ContextEngine.compact]
    I --> J[LLM 生成摘要]
    J --> K[替换历史消息]
    K --> L[保存压缩后会话]
```

---

## 8. 插件加载流程

### 8.1 插件发现与加载序列图

```mermaid
sequenceDiagram
    participant Config as 配置
    participant Discovery as 插件发现器
    participant Loader as 插件加载器
    participant Runtime as 运行时
    participant Plugin as 插件代码

    Config->>Discovery: discoverPlugins()

    par 并行扫描 4 个目录
        Discovery->>Discovery: 1. 配置路径
        Discovery->>Discovery: 2. 工作区扩展
        Discovery->>Discovery: 3. 全局扩展
        Discovery->>Discovery: 4. 捆绑扩展
    end

    Discovery->>Discovery: 合并结果

    loop 每个发现的插件
        Discovery->>Discovery: 读取 openclaw.plugin.json
        Discovery->>Discovery: 验证 manifest Schema
        Discovery->>Discovery: 安全检查路径
    end

    Discovery-->>Loader: 返回插件列表

    loop 每个插件
        Loader->>Loader: 创建 jiti 加载器
        Loader->>Plugin: require(entryPoint)
        Plugin-->>Loader: 返回模块

        Loader->>Loader: 创建 PluginAPI

        alt 函数导出
            Loader->>Plugin: default(api)
        else 对象导出
            Loader->>Plugin: default.register(api)
        end

        Plugin->>Runtime: 注册工具/渠道/命令等
    end

    Loader-->>Config: 插件加载完成
```

### 8.2 插件 API 注册流程

```typescript
// src/plugins/runtime/index.ts

class PluginRuntimeAPI {
  private tools = new Map<string, ToolDefinition>();
  private channels = new Map<string, ChannelPlugin>();
  private routes = new Map<string, HttpRoute>();
  private hooks = new Map<string, HookHandler>();

  registerTool(tool: ToolDefinition): void {
    // 验证工具 Schema
    validateToolDefinition(tool);

    // 检查命名冲突
    if (this.tools.has(tool.name)) {
      throw new Error(`Tool already registered: ${tool.name}`);
    }

    this.tools.set(tool.name, tool);
    this.gateway.tools.register(tool);
    this.logger.info(`Plugin registered tool: ${tool.name}`);
  }

  registerChannel(channel: ChannelPlugin): void {
    validateChannelPlugin(channel);

    // 检查渠道 ID 冲突
    if (this.channels.has(channel.id)) {
      throw new Error(`Channel already registered: ${channel.id}`);
    }

    this.channels.set(channel.id, channel);
    this.gateway.channels.registerPlugin(channel);
    this.logger.info(`Plugin registered channel: ${channel.id}`);
  }

  registerHttpRoute(route: HttpRoute): void {
    // 检查路由冲突
    const existing = this.gateway.http.getRoute(route.path);
    if (existing && !route.replaceExisting) {
      throw new Error(
        `Route conflict at ${route.path}. ` +
        `Use replaceExisting: true to override.`
      );
    }

    this.gateway.http.registerRoute(route);
    this.logger.info(`Plugin registered route: ${route.path}`);
  }

  on(hookName: string, handler: HookHandler, options?: { priority?: number }): void {
    this.hooks.set(`${hookName}:${handler.name}`, {
      hookName,
      handler,
      priority: options?.priority ?? 0,
    });
    this.gateway.hooks.register({
      phase: 'before',
      target: hookName,
      handler,
      priority: options?.priority ?? 0,
    });
  }
}
```

### 8.3 插件依赖加载流程图

```mermaid
flowchart TD
    A[开始加载插件] --> B[读取 package.json]

    B --> C{有依赖？}
    C -->|否 | D[直接加载入口]
    C -->|是 | E[检查 node_modules]

    E --> F{依赖已安装？}
    F -->|否 | G[运行 npm install]
    F -->|是 | D

    G --> H{安装成功？}
    H -->|是 | D
    H -->|否 | I[记录错误]

    I --> J[跳过该插件]

    D --> K[jiti 加载 TypeScript]
    K --> L{加载成功？}
    L -->|是 | M[调用插件注册函数]
    L -->|否 | I

    M --> N[注册完成]
```

---

## 9. 错误处理与恢复

### 9.1 WebSocket 重连流程

```mermaid
sequenceDiagram
    participant Client as GatewayClient
    participant Gateway as Gateway Server

    Client->>Gateway: WebSocket 连接
    Gateway-->>Client: 连接建立

    Note over Client,Gateway: 正常运行中...

    Gateway-->>Client: 连接意外断开 (code: 1006)

    Client->>Client: 清空 pending 请求
    Client->>Client: 指数退避 (1s, 2s, 4s...)

    Client->>Gateway: 尝试重连 #1
    Gateway-->>Client: 失败

    Client->>Client: 等待 2s
    Client->>Gateway: 尝试重连 #2
    Gateway-->>Client: 成功

    Client->>Gateway: req:connect (带 deviceToken)
    Gateway->>Gateway: 验证设备令牌
    Gateway-->>Client: res:connect (hello-ok)

    Client->>Client: 重置退避计数器
    Client->>Client: 恢复事件订阅
```

### 9.2 渠道错误恢复策略

```typescript
// src/channels/channel-health-monitor.ts

interface ChannelHealthConfig {
  maxRetries: number;           // 最大重试次数：5
  retryDelayMs: number;         // 重试间隔：5000
  circuitBreakerThreshold: number; // 熔断阈值：10
  circuitBreakerTimeoutMs: number; // 熔断超时：60000
}

class ChannelHealthMonitor {
  private state = new Map<string, ChannelState>();

  async reportError(channelId: string, error: Error): Promise<void> {
    let state = this.state.get(channelId);

    if (!state) {
      state = { failures: 0, lastFailure: Date.now(), circuitOpen: false };
      this.state.set(channelId, state);
    }

    state.failures += 1;
    state.lastFailure = Date.now();

    // 检查是否触发熔断
    if (state.failures >= this.config.circuitBreakerThreshold) {
      state.circuitOpen = true;
      this.logger.warn(`Channel ${channelId} circuit breaker opened`);

      // 设置超时恢复
      setTimeout(() => {
        state.circuitOpen = false;
        state.failures = 0;
        this.logger.info(`Channel ${channelId} circuit breaker closed`);
      }, this.config.circuitBreakerTimeoutMs);
    }

    // 通知渠道重启
    if (!state.circuitOpen) {
      await this.scheduleRetry(channelId);
    }
  }

  private async scheduleRetry(channelId: string): Promise<void> {
    const state = this.state.get(channelId);
    const delay = Math.min(
      this.config.retryDelayMs * Math.pow(2, state?.failures ?? 0),
      300000  // 最大 5 分钟
    );

    await this.sleep(delay);
    await this.gateway.channels.restart(channelId);
  }
}
```

---

## 10. 监控与日志

### 10.1 日志层级结构

```mermaid
flowchart TB
    A[日志系统] --> B[Subsystem Logger]

    B --> C[gateway]
    B --> D[channels]
    B --> E[agents]
    B --> F[plugins]
    B --> G[auth]

    C --> C1[connection]
    C --> C2[protocol]
    C --> C3[auth]

    D --> D1[telegram]
    D --> D2[discord]
    D --> D3[slack]

    E --> E1[agent-loop]
    E --> E2[tools]
    E --> E3[sessions]

    F --> F1[discovery]
    F --> F2[loader]
    F --> F3[runtime]
```

### 10.2 日志输出格式

```typescript
// src/logging.ts

interface LogEntry {
  timestamp: string;      // ISO 8601
  level: 'debug' | 'info' | 'warn' | 'error';
  subsystem: string;      // e.g., 'gateway/connection'
  message: string;
  context?: Record<string, unknown>;
  stack?: string;         // 仅 error 级别
}

// 示例输出
/*
2026-03-10T14:30:45.123Z [INFO] [gateway] WebSocket server started on 127.0.0.1:18789
2026-03-10T14:30:46.456Z [INFO] [channels/telegram] Connected to Telegram API
2026-03-10T14:31:00.789Z [DEBUG] [gateway/connection] New connection from 127.0.0.1
2026-03-10T14:31:01.012Z [WARN] [auth] Failed login attempt from 192.168.1.100
2026-03-10T14:31:02.345Z [ERROR] [channels/discord] Gateway connection failed
  Error: Rate limited
    at DiscordClient.connect (...)
    ...
*/
```

---

## 附录：缩写与术语

| 缩写 | 含义 |
|------|------|
| DM | Direct Message (私聊) |
| RPC | Remote Procedure Call |
| TTL | Time To Live |
| LRU | Least Recently Used |
| HMAC | Hash-based Message Authentication Code |
| E2E | End-to-End |

---

*文档版本：1.0*
*OpenClaw 版本：2026.3.9*
