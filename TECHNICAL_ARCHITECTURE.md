# OpenClaw 技术架构与实现原理

> 深入解析 OpenClaw 的内部实现机制
> 版本：2026.3.9
> 最后更新：2026-03-10

---

## 目录

1. [系统整体架构](#1-系统整体架构)
2. [核心模块实现](#2-核心模块实现)
3. [通信协议详解](#3-通信协议详解)
4. [数据存储机制](#4-数据存储机制)
5. [插件系统实现](#5-插件系统实现)
6. [安全机制实现](#6-安全机制实现)
7. [运行时管理](#7-运行时管理)
8. [扩展开发指南](#8-扩展开发指南)

---

## 1. 系统整体架构

### 1.1 三层架构设计

OpenClaw 采用清晰的三层架构设计：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           应用层 (Application Layer)                     │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │    CLI      │  │   Web UI    │  │  macOS App  │  │  Mobile App │     │
│  │  命令行工具  │  │  网页控制台  │  │  菜单栏应用  │  │  iOS/Android│     │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘     │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ WebSocket (端口 18789)
                                    │
┌─────────────────────────────────────────────────────────────────────────┐
│                           控制层 (Control Layer)                         │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                    Gateway Core (网关核心)                       │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │    │
│  │  │  Connection  │  │   Session    │  │     Tool     │          │    │
│  │  │   Manager    │  │   Manager    │  │   Executor   │          │    │
│  │  │  连接管理器   │  │   会话管理器  │  │   工具执行器  │          │    │
│  │  └──────────────┘  └──────────────┘  └──────────────┘          │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │    │
│  │  │   Plugin     │  │    Config    │  │    Event     │          │    │
│  │  │   Runtime    │  │    Manager   │  │    Bus       │          │    │
│  │  │  插件运行时   │  │   配置管理器  │  │   事件总线    │          │    │
│  │  └──────────────┘  └──────────────┘  └──────────────┘          │    │
│  └─────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ 内部调用 / RPC
                                    │
┌─────────────────────────────────────────────────────────────────────────┐
│                           基础设施层 (Infrastructure Layer)              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │   Channel   │  │   Model     │  │   Memory    │  │   System    │    │
│  │  Adapters   │  │  Providers  │  │   Store     │  │   Tools     │    │
│  │  渠道适配器  │  │  模型提供者  │  │   存储层    │  │   系统工具  │    │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.2 进程模型

```
┌─────────────────────────────────────────────────────────────┐
│                      主进程 (Main Process)                   │
│  ┌───────────────────────────────────────────────────────┐  │
│  │                    Gateway Server                      │  │
│  │  - WebSocket 服务器 (18789)                            │  │
│  │  - HTTP 服务器 (静态资源/API)                           │  │
│  │  - 渠道连接管理                                         │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
         │
         │ 派生
         ▼
┌─────────────────────────────────────────────────────────────┐
│                    Agent 进程 (隔离执行)                      │
│  ┌───────────────────────────────────────────────────────┐  │
│  │                    Pi Agent Runtime                    │  │
│  │  - 提示构建与模型调用                                   │  │
│  │  - 工具执行协调                                         │  │
│  │  - 会话上下文管理                                       │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### 1.3 核心文件结构

```
src/
├── entry.ts                 # 程序入口点
├── gateway/                 # 网关核心
│   ├── gateway.ts           # 网关主类
│   ├── client.ts            # WebSocket 客户端
│   ├── net.ts               # 网络工具
│   ├── protocol/            # 协议定义
│   │   └── index.ts         # TypeBox Schema 定义
│   ├── auth.ts              # 认证逻辑
│   ├── connection-auth.ts   # 连接认证
│   └── device-auth.ts       # 设备认证
├── agents/                  # Agent 运行时
│   ├── agent-loop.ts        # Agent 循环
│   ├── session-loop.ts      # 会话循环
│   ├── auth-profiles.ts     # 认证配置管理
│   ├── bash-tools.ts        # Bash 工具实现
│   └── tools/               # 工具实现
├── channels/                # 渠道层
│   ├── plugins/             # 渠道插件
│   ├── onboarding/          # 渠道引导
│   └── normalize/           # 消息标准化
├── config/                  # 配置系统
│   ├── io.ts                # 配置 IO
│   ├── validation.ts        # 配置验证
│   └── sessions/            # 会话存储
│       ├── store.ts         # 会话存储实现
│       └── types.ts         # 会话类型定义
├── plugins/                 # 插件系统
│   ├── runtime/             # 插件运行时
│   ├── loader.ts            # 插件加载器
│   └── registry.ts          # 插件注册表
├── tools/                   # 工具实现
│   ├── browser/             # 浏览器工具
│   ├── canvas/              # Canvas 工具
│   └── media/               # 媒体工具
└── infra/                   # 基础设施
    ├── ports.ts             # 端口管理
    ├── device-identity.ts   # 设备身份
    └── tls/                 # TLS 工具
```

---

## 2. 核心模块实现

### 2.1 Gateway 核心实现

#### 2.1.1 Gateway 初始化流程

```typescript
// src/gateway/gateway.ts (简化版)

class Gateway {
  private config: Config;
  private channels: Map<string, Channel>;
  private clients: Map<string, GatewayClient>;
  private sessions: SessionStore;

  async start(options: GatewayOptions): Promise<void> {
    // 1. 加载配置
    this.config = loadConfig();

    // 2. 初始化会话存储
    this.sessions = await createSessionStore(this.config);

    // 3. 启动 WebSocket 服务器
    await this.startWebSocketServer({
      bind: this.config.gateway.bind,
      port: this.config.gateway.port,
    });

    // 4. 加载并初始化渠道
    await this.loadChannels(this.config.channels);

    // 5. 加载并初始化插件
    await this.loadPlugins(this.config.plugins);

    // 6. 启动后台服务
    await this.startServices();

    log.info(`Gateway started on ${this.config.gateway.bind}:${this.config.gateway.port}`);
  }
}
```

#### 2.1.2 WebSocket 连接处理

```typescript
// src/gateway/client.ts (核心逻辑)

class GatewayClient {
  private ws: WebSocket | null = null;
  private pending = new Map<string, Pending>();
  private lastSeq: number | null = null;

  start() {
    const url = this.opts.url ?? "ws://127.0.0.1:18789";

    // 安全检查：阻止非 loopback 的 ws:// 连接
    if (!isSecureWebSocketUrl(url, { allowPrivateWs })) {
      this.opts.onConnectError?.(new Error(
        "SECURITY ERROR: Cannot connect to non-loopback over plaintext ws://"
      ));
      return;
    }

    this.ws = new WebSocket(url, {
      maxPayload: 25 * 1024 * 1024,  // 25MB 最大负载
    });

    this.ws.on("open", () => {
      if (this.opts.tlsFingerprint) {
        this.validateTlsFingerprint();
      }
      this.queueConnect();  // 发送连接握手
    });

    this.ws.on("message", (data) => this.handleMessage(rawDataToString(data)));
    this.ws.on("close", (code, reason) => {
      this.ws = null;
      this.scheduleReconnect();  // 自动重连
    });
  }

  private async sendConnect() {
    // 构建设备认证载荷
    const authPayload = buildDeviceAuthPayloadV3({
      deviceId: this.opts.deviceIdentity.deviceId,
      platform: this.opts.platform,
      deviceFamily: this.opts.deviceFamily,
      nonce: this.connectNonce,  // 来自 Gateway 的挑战
    });

    // 使用设备私钥签名
    const signedPayload = await signDevicePayload(
      authPayload,
      this.opts.deviceIdentity.privateKey
    );

    // 发送连接请求
    this.sendRequest("connect", {
      device: {
        id: this.opts.deviceIdentity.deviceId,
        platform: this.opts.platform,
        name: this.opts.clientDisplayName,
      },
      auth: {
        token: this.opts.token,
        deviceToken: this.opts.deviceToken,
        signature: signedPayload.signature,
      },
    });
  }
}
```

### 2.2 会话管理系统

#### 2.2.1 会话键派生

```typescript
// src/config/sessions/session-key.ts

/**
 * 从渠道信息派生唯一的会话键
 *
 * 算法：
 * 1. 规范化渠道类型和账号 ID
 * 2. 组合渠道信息和对端 ID
 * 3. 生成 SHA256 哈希作为会话键
 */
export function deriveSessionKey(params: {
  channelType: string;
  accountId: string;
  peerId: string;
  groupId?: string;
}): string {
  const { channelType, accountId, peerId, groupId } = params;

  // 规范化输入
  const normalizedChannel = channelType.trim().toLowerCase();
  const normalizedAccount = accountId.trim().toLowerCase();
  const normalizedPeer = peerId.trim();

  // 组合键
  const keyParts = [
    normalizedChannel,
    normalizedAccount,
    groupId ? `group:${groupId}` : normalizedPeer,
  ];

  // 生成哈希
  const hash = crypto.createHash("sha256");
  hash.update(keyParts.join("::"));

  return hash.digest("hex").slice(0, 32);  // 截取前 32 字符
}
```

#### 2.2.2 会话存储实现

```typescript
// src/config/sessions/store.ts

interface SessionEntry {
  channel: string;           // 渠道类型
  lastChannel: string;       // 最后使用的渠道
  lastTo: string;            // 最后发送的目标
  lastAccountId: string;     // 最后使用的账号
  lastThreadId?: string;     // 最后使用的线程 ID
  deliveryContext?: DeliveryContext;
  modelOverride?: string;    // 模型覆盖
  providerOverride?: string; // 提供者覆盖
  createdAt: number;         // 创建时间戳
  updatedAt: number;         // 更新时间戳
  activity: SessionActivity; // 活动记录
}

class SessionStore {
  private store: Record<string, SessionEntry> = {};
  private storePath: string;
  private cacheTtlMs: number = 45_000;  // 45 秒缓存 TTL

  async loadSessionStore(): Promise<Record<string, SessionEntry>> {
    // 检查缓存
    if (this.isCacheEnabled()) {
      const cached = await this.readCache();
      if (cached) {
        return cached;
      }
    }

    // 从磁盘加载
    const content = await fs.readFile(this.storePath, "utf-8");
    this.store = JSON.parse(content);

    // 规范化会话条目
    this.normalizeSessionStore();

    return this.store;
  }

  async saveSessionEntry(
    sessionKey: string,
    entry: SessionEntry
  ): Promise<void> {
    // 获取写锁
    await this.withSessionStoreLock(async () => {
      // 规范化会话键
      const normalizedKey = normalizeStoreSessionKey(sessionKey);

      // 规范化交付上下文
      const normalizedEntry = normalizeSessionEntryDelivery(entry);

      // 更新存储
      this.store[normalizedKey] = {
        ...this.store[normalizedKey],
        ...normalizedEntry,
        updatedAt: Date.now(),
      };

      // 写入磁盘
      await this.writeStoreFile();

      // 更新缓存
      this.updateCache();
    });
  }

  private async withSessionStoreLock<T>(
    fn: () => Promise<T>
  ): Promise<T> {
    // 实现互斥锁，防止并发写入
    const queue = this.getLockQueue(this.storePath);
    return await queue.enqueue(fn);
  }
}
```

### 2.3 Agent 循环实现

#### 2.3.1 Agent 执行流程

```typescript
// src/agents/agent-loop.ts (核心逻辑)

interface AgentLoopParams {
  sessionKey: string;
  message: string;
  thinking: ThinkingLevel;
  verbose: boolean;
  model?: string;
  provider?: string;
}

class AgentLoop {
  private session: Session;
  private tools: Map<string, Tool>;
  private contextEngine: ContextEngine;

  async execute(params: AgentLoopParams): Promise<AgentResult> {
    // 1. 加载会话上下文
    const session = await this.loadSession(params.sessionKey);

    // 2. 应用钩子 (before_model_resolve)
    const modelOverride = await this.runBeforeModelResolveHooks({
      session,
      message: params.message,
    });

    // 3. 解析模型
    const model = params.model ?? modelOverride ?? session.modelOverride;
    const provider = this.resolveProvider(model);

    // 4. 构建提示
    const prompt = await this.contextEngine.assemble({
      session,
      message: params.message,
      systemPrompt: this.buildSystemPrompt(session),
    });

    // 5. 调用模型 (流式)
    const stream = await provider.completeStream(prompt);

    // 6. 处理流式响应
    const result = await this.processStream(stream, {
      onChunk: (chunk) => this.sendToChannel(chunk),
      onToolCall: (toolCall) => this.executeTool(toolCall),
    });

    // 7. 更新会话上下文
    await this.contextEngine.ingest({
      session,
      message: params.message,
      response: result.text,
      toolCalls: result.toolCalls,
    });

    // 8. 检查是否需要压缩
    if (this.shouldCompact(session)) {
      await this.contextEngine.compact(session);
    }

    return result;
  }

  private async executeTool(toolCall: ToolCall): Promise<ToolResult> {
    const tool = this.tools.get(toolCall.name);
    if (!tool) {
      throw new Error(`Unknown tool: ${toolCall.name}`);
    }

    // 检查工具权限
    if (!this.isToolAllowed(toolCall)) {
      // 请求批准
      const approved = await this.requestApproval(toolCall);
      if (!approved) {
        return { error: "Tool execution denied" };
      }
    }

    // 执行工具
    return await tool.execute(toolCall.input, {
      session: this.session,
      config: this.config,
    });
  }
}
```

### 2.4 渠道层实现

#### 2.4.1 渠道适配器接口

```typescript
// src/channels/plugins/index.ts

interface ChannelPlugin {
  id: string;
  meta: {
    id: string;
    label: string;
    selectionLabel: string;
    docsPath: string;
    blurb: string;
    aliases?: string[];
  };

  capabilities: {
    chatTypes: ("direct" | "group")[];
    mediaTypes?: ("image" | "video" | "audio")[];
    supportsThreads?: boolean;
    supportsReactions?: boolean;
  };

  config: {
    listAccountIds: (cfg: Config) => string[];
    resolveAccount: (cfg: Config, accountId?: string) => ChannelAccount;
    inspectAccount?: (cfg: Config, accountId: string) => AccountSnapshot;
  };

  outbound: {
    deliveryMode: "direct" | "broadcast";
    sendText: (params: SendTextParams) => Promise<SendResult>;
    sendMedia?: (params: SendMediaParams) => Promise<SendResult>;
  };

  // 可选：渠道特定功能
  gateway?: {
    start: (cfg: Config, accountId: string) => Promise<void>;
    stop: (accountId: string) => Promise<void>;
  };

  actions?: {
    reply?: (params: ActionParams) => Promise<void>;
    react?: (params: ActionParams) => Promise<void>;
    delete?: (params: ActionParams) => Promise<void>;
  };
}
```

#### 2.4.2 消息标准化

```typescript
// src/channels/plugins/normalize/shared.ts

interface NormalizedMessage {
  id: string;
  channelId: string;
  accountId: string;
  peerId: string;
  threadId?: string;
  sender: {
    id: string;
    name: string;
    isBot: boolean;
  };
  content: {
    type: "text" | "image" | "video" | "audio";
    text?: string;
    mediaUrl?: string;
    mimeType?: string;
  };
  timestamp: number;
  isIncoming: boolean;
}

/**
 * 将渠道特定消息格式标准化为统一格式
 */
function normalizeMessage(
  rawMessage: unknown,
  channelType: string
): NormalizedMessage {
  switch (channelType) {
    case "telegram":
      return normalizeTelegramMessage(rawMessage);
    case "discord":
      return normalizeDiscordMessage(rawMessage);
    case "slack":
      return normalizeSlackMessage(rawMessage);
    case "whatsapp":
      return normalizeWhatsappMessage(rawMessage);
    // ... 其他渠道
    default:
      throw new Error(`Unknown channel type: ${channelType}`);
  }
}
```

---

## 3. 通信协议详解

### 3.1 WebSocket 协议格式

#### 3.1.1 帧结构

```typescript
// src/gateway/protocol/index.ts

// 协议版本号
const PROTOCOL_VERSION = 3;

// 请求帧
interface RequestFrame {
  type: "req";
  id: string;        // UUID
  method: string;    // 方法名
  params: unknown;   // 参数
}

// 响应帧
interface ResponseFrame {
  type: "res";
  id: string;        // 对应请求 ID
  ok: boolean;
  payload?: unknown;
  error?: ErrorShape;
}

// 事件帧
interface EventFrame {
  type: "event";
  event: string;     // 事件类型
  payload: unknown;
  seq?: number;      // 序列号 (用于检测丢失)
  stateVersion?: number;
}

// 错误形状
interface ErrorShape {
  code: string;
  message: string;
  details?: Record<string, unknown>;
}
```

#### 3.1.2 连接握手流程

```
客户端                              Gateway
  │                                   │
  │────── req: connect ─────────────> │
  │        {                          │
  │          device: {...},           │
  │          auth: {...}              │
  │        }                          │
  │                                   │
  │<───── res: connect ────────────── │
  │        {                          │
  │          type: "hello-ok",        │
  │          snapshot: {              │
  │            presence: {...},       │
  │            health: {...}          │
  │          }                        │
  │        }                          │
  │                                   │
  │<───── event: presence ──────────> │
  │<───── event: tick ──────────────> │  (每 30 秒)
```

### 3.2 设备认证协议

#### 3.2.1 配对流程

```typescript
// src/gateway/device-auth.ts

interface DeviceAuthPayloadV3 {
  deviceId: string;
  platform: string;
  deviceFamily: string;
  nonce: string;       // Gateway 生成的挑战
  timestamp: number;
}

/**
 * 构建设备认证载荷
 */
function buildDeviceAuthPayloadV3(params: {
  deviceId: string;
  platform: string;
  deviceFamily: string;
  nonce: string;
}): DeviceAuthPayloadV3 {
  return {
    deviceId: params.deviceId,
    platform: params.platform,
    deviceFamily: params.deviceFamily,
    nonce: params.nonce,
    timestamp: Date.now(),
  };
}

/**
 * 使用设备私钥签名载荷
 */
async function signDevicePayload(
  payload: DeviceAuthPayloadV3,
  privateKey: string
): Promise<{ payload: string; signature: string }> {
  const payloadJson = JSON.stringify(payload);
  const payloadBytes = new TextEncoder().encode(payloadJson);

  const signature = await crypto.subtle.sign(
    "EdDSA",
    privateKey,
    payloadBytes
  );

  return {
    payload: payloadJson,
    signature: arrayBufferToBase64Url(signature),
  };
}
```

#### 3.2.2 配对码生成

```typescript
// src/gateway/auth.ts

/**
 * 生成 6 位数字配对码
 */
function generatePairingCode(): string {
  const code = crypto.randomInt(100000, 999999);
  return code.toString();
}

/**
 * 验证配对码 (带过期时间)
 */
class PairingCodeVerifier {
  private codes = new Map<string, {
    code: string;
    channel: string;
    deviceId: string;
    expiresAt: number;
  }>();

  generate(params: {
    channel: string;
    deviceId: string;
  }): string {
    const code = generatePairingCode();
    this.codes.set(code, {
      code,
      channel: params.channel,
      deviceId: params.deviceId,
      expiresAt: Date.now() + 5 * 60 * 1000,  // 5 分钟过期
    });
    return code;
  }

  verify(code: string): { channel: string; deviceId: string } | null {
    const entry = this.codes.get(code);
    if (!entry) {
      return null;
    }
    if (Date.now() > entry.expiresAt) {
      this.codes.delete(code);
      return null;
    }
    this.codes.delete(code);
    return {
      channel: entry.channel,
      deviceId: entry.deviceId,
    };
  }
}
```

### 3.3 事件系统

#### 3.3.1 事件类型定义

```typescript
// src/gateway/protocol/index.ts

// Agent 事件 (流式响应)
interface AgentEvent {
  runId: string;
  chunk:
    | { type: "text"; text: string }
    | { type: "tool_call"; toolCall: ToolCall }
    | { type: "tool_result"; result: ToolResult }
    | { type: "done"; summary: string };
}

// 聊天事件
interface ChatEvent {
  channel: string;
  accountId: string;
  peerId: string;
  message: NormalizedMessage;
}

// 存在事件
interface PresenceEvent {
  channel: string;
  accountId: string;
  status: "online" | "offline" | "connecting";
  lastSeen?: number;
}

// 健康事件
interface HealthEvent {
  status: "healthy" | "degraded" | "unhealthy";
  channels: Record<string, ChannelHealth>;
  memory: MemoryHealth;
  uptime: number;
}
```

#### 3.3.2 事件序列管理

```typescript
// src/gateway/events.ts

class EventDispatcher {
  private seq = 0;
  private subscribers = new Map<string, Set<EventSubscriber>>();

  subscribe(
    clientId: string,
    eventTypes: string[],
    handler: (event: EventFrame) => void
  ): void {
    const subscriber = { eventTypes, handler };
    let subscribers = this.subscribers.get(clientId);
    if (!subscribers) {
      subscribers = new Set();
      this.subscribers.set(clientId, subscribers);
    }
    subscribers.add(subscriber);
  }

  emit(eventType: string, payload: unknown): void {
    this.seq += 1;

    const frame: EventFrame = {
      type: "event",
      event: eventType,
      payload,
      seq: this.seq,
    };

    for (const [clientId, subscribers] of this.subscribers) {
      for (const subscriber of subscribers) {
        if (subscriber.eventTypes.includes(eventType) ||
            subscriber.eventTypes.includes("*")) {
          subscriber.handler(frame);
        }
      }
    }
  }
}
```

---

## 4. 数据存储机制

### 4.1 会话存储

#### 4.1.1 存储结构

```typescript
// src/config/sessions/types.ts

interface SessionStore {
  // 会话条目映射
  [sessionKey: string]: SessionEntry;
}

interface SessionEntry {
  // 渠道信息
  channel: string;
  lastChannel: string;
  lastTo: string;
  lastAccountId: string;
  lastThreadId?: string;

  // 交付上下文
  deliveryContext?: DeliveryContext;

  // 模型配置
  modelOverride?: string;
  providerOverride?: string;
  thinkingLevel?: ThinkingLevel;
  verboseLevel?: VerboseLevel;

  // 时间戳
  createdAt: number;
  updatedAt: number;
  lastActivityAt: number;

  // 活动记录
  activity: SessionActivity;

  // 组信息 (如果是群聊)
  group?: GroupInfo;

  // 工件 (附件)
  artifacts?: Artifact[];
}

interface DeliveryContext {
  channel: string;
  to: string;
  accountId: string;
  threadId?: string;
}

interface SessionActivity {
  messageCount: number;
  tokenCount: number;
  lastModelCall?: number;
  lastToolCall?: ToolCallRecord;
}
```

#### 4.1.2 存储维护

```typescript
// src/config/sessions/store-maintenance.ts

interface MaintenanceConfig {
  maxEntries: number;           // 最大会话数
  maxAgeDays: number;           // 最大保留天数
  pruneStaleAfterDays: number;  // 清理不活跃会话的天数
}

/**
 * 维护会话存储 (定期清理)
 */
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

#### 4.1.3 磁盘预算控制

```typescript
// src/config/sessions/disk-budget.ts

interface DiskBudgetConfig {
  maxTotalBytes: number;      // 总磁盘预算
  warningThreshold: number;   // 警告阈值 (0-1)
  sweepStrategy: "oldest" | "largest" | "lru";
}

/**
 * 执行磁盘预算清理
 */
async function enforceSessionDiskBudget(
  store: SessionStore,
  storePath: string,
  config: DiskBudgetConfig
): Promise<DiskBudgetSweepResult> {
  // 获取当前磁盘使用量
  const currentUsage = await getDirectorySize(storePath);

  if (currentUsage <= config.maxTotalBytes) {
    return { swept: 0, bytesFreed: 0, reason: "within_budget" };
  }

  // 计算需要释放的空间
  const targetUsage = config.maxTotalBytes * config.warningThreshold;
  const bytesToFree = currentUsage - targetUsage;

  // 根据策略排序会话
  const sessions = await getSessionSizes(storePath, store);
  const sorted = sortSessions(sessions, config.sweepStrategy);

  // 删除会话直到达到目标
  let bytesFreed = 0;
  let swept = 0;

  for (const session of sorted) {
    if (bytesFreed >= bytesToFree) {
      break;
    }

    await deleteSessionFile(session.key, storePath);
    delete store[session.key];

    bytesFreed += session.size;
    swept++;
  }

  return {
    swept,
    bytesFreed,
    reason: "budget_exceeded",
  };
}
```

### 4.2 配置存储

#### 4.2.1 配置层次

```typescript
// src/config/io.ts

interface ConfigSource {
  path: string;
  data: Record<string, unknown>;
  priority: number;  // 优先级 (高优先级覆盖低优先级)
}

/**
 * 加载配置 (多层合并)
 */
async function loadConfig(): Promise<Config> {
  const sources: ConfigSource[] = [
    // 1. 默认配置 (最低优先级)
    {
      path: getDefaultsPath(),
      data: DEFAULT_CONFIG,
      priority: 0,
    },
    // 2. 全局配置
    {
      path: getGlobalConfigPath(),
      data: await readConfigFile(getGlobalConfigPath()),
      priority: 1,
    },
    // 3. 工作区配置
    {
      path: getWorkspaceConfigPath(),
      data: await readConfigFile(getWorkspaceConfigPath()),
      priority: 2,
    },
    // 4. 环境变量 (最高优先级)
    {
      path: "env",
      data: parseEnvConfig(process.env),
      priority: 3,
    },
  ];

  // 合并配置
  const merged = mergeConfigs(sources);

  // 验证配置
  await validateConfigObjectWithPlugins(merged);

  return merged as Config;
}
```

#### 4.2.2 配置验证

```typescript
// src/config/validation.ts

import { Type } from "@sinclair/typebox";
import { TypeCompiler } from "@sinclair/typebox/compiler";

// 使用 TypeBox 定义配置 Schema
const ConfigSchema = Type.Object({
  gateway: Type.Object({
    bind: Type.String(),
    port: Type.Number(),
    auth: Type.Object({
      mode: Type.Union([
        Type.Literal("none"),
        Type.Literal("token"),
        Type.Literal("password"),
      ]),
      token: Type.Optional(Type.String()),
    }),
  }),
  models: Type.Object({
    default: Type.String(),
    providers: Type.Record(Type.String(), Type.Any()),
  }),
  channels: Type.Object({
    // 动态字段，由插件扩展
  }, { additionalProperties: true }),
  plugins: Type.Object({
    enabled: Type.Boolean(),
    entries: Type.Record(Type.String(), Type.Any()),
  }),
});

// 编译验证器
const ConfigValidator = TypeCompiler.Compile(ConfigSchema);

/**
 * 验证配置对象
 */
function validateConfigObject(
  config: Record<string, unknown>
): ValidationResult {
  const errors = [...ConfigValidator.Errors(config)];

  if (errors.length > 0) {
    return {
      valid: false,
      errors: errors.map(e => ({
        path: e.path,
        message: e.message,
      })),
    };
  }

  return { valid: true };
}
```

---

## 5. 插件系统实现

### 5.1 插件发现与加载

#### 5.1.1 插件发现流程

```typescript
// src/plugins/discovery.ts

interface PluginDiscoveryResult {
  plugins: DiscoveredPlugin[];
  errors: DiscoveryError[];
}

interface DiscoveredPlugin {
  id: string;
  path: string;
  source: "config" | "workspace" | "global" | "bundled";
  manifest: PluginManifest;
  entryPoint: string;
}

/**
 * 发现所有可用插件
 */
async function discoverPlugins(config: Config): Promise<PluginDiscoveryResult> {
  const plugins: DiscoveredPlugin[] = [];
  const errors: DiscoveryError[] = [];

  // 1. 从配置路径发现
  if (config.plugins.load?.paths) {
    for (const path of config.plugins.load.paths) {
      const result = await discoverPluginAtPath(path, "config");
      if (result.ok) {
        plugins.push(result.plugin);
      } else {
        errors.push(result.error);
      }
    }
  }

  // 2. 从工作区扩展发现
  const workspaceExtensions = await findWorkspaceExtensions();
  for (const ext of workspaceExtensions) {
    const result = await discoverPluginAtPath(ext, "workspace");
    if (result.ok) {
      plugins.push(result.plugin);
    }
  }

  // 3. 从全局扩展发现
  const globalExtensions = await findGlobalExtensions();
  for (const ext of globalExtensions) {
    const result = await discoverPluginAtPath(ext, "global");
    if (result.ok) {
      plugins.push(result.plugin);
    }
  }

  // 4. 从捆绑扩展发现
  const bundledExtensions = await findBundledExtensions();
  for (const ext of bundledExtensions) {
    const result = await discoverPluginAtPath(ext, "bundled");
    if (result.ok) {
      plugins.push(result.plugin);
    }
  }

  // 5. 去重 (按 ID，优先级高的保留)
  const deduped = deduplicatePlugins(plugins);

  return { plugins: deduped, errors };
}
```

#### 5.1.2 插件加载器

```typescript
// src/plugins/loader.ts

import jiti from "jiti";

interface PluginAPI {
  registerTool: (tool: ToolDefinition) => void;
  registerChannel: (channel: ChannelPlugin) => void;
  registerCli: (fn: (program: Command) => void) => void;
  registerHttpRoute: (route: HttpRoute) => void;
  registerService: (service: ServiceDefinition) => void;
  registerProvider: (provider: ProviderDefinition) => void;
  registerCommand: (command: CommandDefinition) => void;
  on: (hook: string, handler: HookHandler) => void;
  logger: Logger;
  config: Config;
  runtime: RuntimeAPI;
}

/**
 * 加载单个插件
 */
async function loadPlugin(
  plugin: DiscoveredPlugin,
  api: PluginAPI
): Promise<LoadedPlugin> {
  // 安全检查：验证插件路径
  await verifyPluginPathSafety(plugin.path);

  // 使用 jiti 加载 TypeScript
  const jitiLoader = jiti(__filename, {
    interopDefault: true,
    esmResolve: true,
  });

  // 加载插件入口
  const module = jitiLoader(plugin.entryPoint) as PluginModule;

  // 获取插件导出
  let pluginExport: PluginExport;
  if (typeof module === "function") {
    pluginExport = { default: module };
  } else if (typeof module.default === "function") {
    pluginExport = module;
  } else {
    return module as LoadedPlugin;
  }

  // 调用插件注册函数
  if (typeof pluginExport.default === "function") {
    await pluginExport.default(api);
  } else if (pluginExport.default?.register) {
    await pluginExport.default.register(api);
  }

  return {
    id: plugin.id,
    path: plugin.path,
    manifest: plugin.manifest,
  };
}
```

### 5.2 插件运行时

#### 5.2.1 运行时 API 实现

```typescript
// src/plugins/runtime/index.ts

class PluginRuntimeAPI implements RuntimeAPI {
  private gateway: Gateway;
  private config: Config;
  private logger: Logger;

  registerTool(tool: ToolDefinition): void {
    // 验证工具 Schema
    validateToolDefinition(tool);

    // 注册到工具注册表
    this.gateway.tools.register(tool);

    this.logger.info(`Plugin registered tool: ${tool.name}`);
  }

  registerChannel(channel: ChannelPlugin): void {
    // 验证渠道配置
    validateChannelPlugin(channel);

    // 注册到渠道管理器
    this.gateway.channels.registerPlugin(channel);

    this.logger.info(`Plugin registered channel: ${channel.id}`);
  }

  registerHttpRoute(route: HttpRoute): void {
    // 检查路由冲突
    const existing = this.gateway.http.getRoute(route.path);
    if (existing && !route.replaceExisting) {
      throw new Error(
        `Route conflict at ${route.path}. ` +
        "Use replaceExisting: true to override."
      );
    }

    // 注册路由
    this.gateway.http.registerRoute(route);

    this.logger.info(`Plugin registered route: ${route.path}`);
  }

  async fetch(input: RequestInfo, init?: RequestInit): Promise<Response> {
    // 代理 HTTP 请求 (带超时)
    const controller = new AbortController();
    const timeout = setTimeout(() => controller.abort(), 30000);

    try {
      return await fetch(input, {
        ...init,
        signal: controller.signal,
      });
    } finally {
      clearTimeout(timeout);
    }
  }
}
```

#### 5.2.2 钩子系统

```typescript
// src/plugins/hooks.ts

type HookPhase = "before" | "after";

interface HookDefinition {
  phase: HookPhase;
  target: string;       // 例如 "agent_start", "prompt_build"
  handler: HookHandler;
  priority: number;     // 高优先级先执行
}

class HookManager {
  private hooks = new Map<string, HookDefinition[]>();

  register(hook: HookDefinition): void {
    const key = `${hook.phase}:${hook.target}`;
    let list = this.hooks.get(key);
    if (!list) {
      list = [];
      this.hooks.set(key, list);
    }
    list.push(hook);
    // 按优先级排序
    list.sort((a, b) => b.priority - a.priority);
  }

  async runBefore(
    target: string,
    context: HookContext
  ): Promise<HookResult> {
    const key = `before:${target}`;
    const hooks = this.hooks.get(key) ?? [];

    const results: HookResult[] = [];
    for (const hook of hooks) {
      const result = await hook.handler(context);
      if (result) {
        results.push(result);
      }
    }

    return mergeHookResults(results);
  }

  async runAfter(
    target: string,
    context: HookContext,
    result: unknown
  ): Promise<void> {
    const key = `after:${target}`;
    const hooks = this.hooks.get(key) ?? [];

    for (const hook of hooks) {
      await hook.handler({ ...context, result });
    }
  }
}

// 预定义钩子点
const HOOK_POINTS = {
  BEFORE_MODEL_RESOLVE: "before:model_resolve",
  BEFORE_PROMPT_BUILD: "before:prompt_build",
  BEFORE_AGENT_START: "before:agent_start",
  AFTER_TOOL_CALL: "after:tool_call",
  AFTER_MESSAGE_SEND: "after:message_send",
} as const;
```

---

## 6. 安全机制实现

### 6.1 认证系统

#### 6.1.1 认证模式

```typescript
// src/gateway/auth-mode-policy.ts

type AuthMode = "none" | "token" | "password";

interface AuthConfig {
  mode: AuthMode;
  token?: string;
  passwordHash?: string;
  allowTailscale?: boolean;
}

/**
 * 验证连接认证
 */
async function validateConnectionAuth(
  params: ConnectParams,
  config: AuthConfig,
  clientIp: string
): Promise<AuthResult> {
  // 1. 检查是否来自 Tailscale
  const isTailscale = isTailscaleAddress(clientIp);
  if (isTailscale && config.allowTailscale !== false) {
    return { ok: true, method: "tailscale" };
  }

  // 2. 根据模式验证
  switch (config.mode) {
    case "none":
      return { ok: true, method: "none" };

    case "token":
      if (!params.auth?.token) {
        return { ok: false, reason: "token_required" };
      }
      if (params.auth.token !== config.token) {
        return { ok: false, reason: "token_invalid" };
      }
      return { ok: true, method: "token" };

    case "password":
      if (!params.auth?.password) {
        return { ok: false, reason: "password_required" };
      }
      const valid = await verifyPassword(
        params.auth.password,
        config.passwordHash!
      );
      if (!valid) {
        return { ok: false, reason: "password_invalid" };
      }
      return { ok: true, method: "password" };

    default:
      return { ok: false, reason: "unknown_auth_mode" };
  }
}
```

#### 6.1.2 认证速率限制

```typescript
// src/gateway/auth-rate-limit.ts

interface RateLimitState {
  attempts: number;
  firstAttemptAt: number;
  lockedUntil?: number;
}

class AuthRateLimiter {
  private state = new Map<string, RateLimitState>();
  private maxAttempts: number = 5;
  private windowMs: number = 5 * 60 * 1000;  // 5 分钟
  private lockoutMs: number = 15 * 60 * 1000; // 15 分钟

  async checkLimit(identifier: string): Promise<RateLimitResult> {
    const now = Date.now();
    let state = this.state.get(identifier);

    // 检查是否已锁定
    if (state?.lockedUntil) {
      if (now < state.lockedUntil) {
        return {
          allowed: false,
          reason: "locked",
          retryAfter: state.lockedUntil - now,
        };
      }
      // 锁定期已过，重置
      state = undefined;
    }

    // 初始化或获取状态
    if (!state) {
      state = { attempts: 0, firstAttemptAt: now };
    }

    // 检查时间窗口
    if (now - state.firstAttemptAt > this.windowMs) {
      state = { attempts: 0, firstAttemptAt: now };
    }

    // 记录尝试
    state.attempts += 1;
    this.state.set(identifier, state);

    // 检查是否超限
    if (state.attempts >= this.maxAttempts) {
      state.lockedUntil = now + this.lockoutMs;
      return {
        allowed: false,
        reason: "rate_limited",
        retryAfter: this.lockoutMs,
      };
    }

    return { allowed: true };
  }

  recordSuccess(identifier: string): void {
    // 认证成功后重置计数
    this.state.delete(identifier);
  }
}
```

### 6.2 设备配对安全

#### 6.2.1 配对状态管理

```typescript
// src/gateway/device-pairing.ts

interface PairingRequest {
  code: string;
  channel: string;
  channelId: string;
  deviceId: string;
  deviceInfo: DeviceInfo;
  requestedAt: number;
  expiresAt: number;
}

class DevicePairingManager {
  private pending = new Map<string, PairingRequest>();
  private paired = new Map<string, PairedDevice>();

  /**
   * 创建配对请求
   */
  async createPairingRequest(params: {
    channel: string;
    deviceId: string;
    deviceInfo: DeviceInfo;
  }): Promise<string> {
    const code = generatePairingCode();

    const request: PairingRequest = {
      code,
      channel: params.channel,
      channelId: this.resolveChannelId(params.channel),
      deviceId: params.deviceId,
      deviceInfo: params.deviceInfo,
      requestedAt: Date.now(),
      expiresAt: Date.now() + 5 * 60 * 1000,  // 5 分钟
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
      return { ok: false, reason: "code_not_found" };
    }

    if (Date.now() > request.expiresAt) {
      this.pending.delete(code);
      return { ok: false, reason: "code_expired" };
    }

    // 生成设备令牌
    const deviceToken = await this.generateDeviceToken(request.deviceId);

    // 存储配对设备
    this.paired.set(request.deviceId, {
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
   * 拒绝配对
   */
  async rejectPairing(code: string): Promise<void> {
    this.pending.delete(code);
  }
}
```

### 6.3 工具执行安全

#### 6.3.1 Bash 工具批准

```typescript
// src/agents/bash-tools.exec-approval-request.ts

interface ExecApprovalRequest {
  command: string;
  reason: string;
  riskLevel: "low" | "medium" | "high";
  details: {
    workingDir: string;
    env: Record<string, string>;
    timeout: number;
  };
}

class ExecApprovalManager {
  private pending = new Map<string, ExecApprovalRequest>();
  private patterns: {
    allow: RegExp[];
    deny: RegExp[];
  };

  /**
   * 检查命令是否需要批准
   */
  async shouldRequireApproval(
    command: string
  ): Promise<{ requiresApproval: boolean; reason?: string }> {
    // 检查拒绝模式
    for (const pattern of this.patterns.deny) {
      if (pattern.test(command)) {
        return { requiresApproval: true, reason: "denied_pattern" };
      }
    }

    // 检查允许模式 (自动批准)
    for (const pattern of this.patterns.allow) {
      if (pattern.test(command)) {
        return { requiresApproval: false };
      }
    }

    // 默认需要批准
    return { requiresApproval: true };
  }

  /**
   * 创建批准请求
   */
  async createApprovalRequest(
    command: string,
    context: ExecContext
  ): Promise<string> {
    const riskLevel = this.assessRisk(command);

    const request: ExecApprovalRequest = {
      command,
      reason: this.getApprovalReason(command),
      riskLevel,
      details: {
        workingDir: context.cwd,
        env: context.env,
        timeout: context.timeout,
      },
    };

    const id = randomUUID();
    this.pending.set(id, request);

    // 发送批准请求到管理员渠道
    await this.sendApprovalRequest(id, request);

    return id;
  }

  /**
   * 评估命令风险等级
   */
  private assessRisk(command: string): "low" | "medium" | "high" {
    const highRiskPatterns = [
      /rm\s+(-[rf]+\s+)?\//,      // 删除根目录
      /sudo\s+/,                   // 提权命令
      /curl.*\|\s*(ba)?sh/,        // 下载并执行
      /wget.*\|\s*(ba)?sh/,
      /chmod\s+[0-7]+777/,         // 设置可执行权限
      /mkfs/,                      // 格式化
      /dd\s+.*of=\/dev/,           // 直接写入设备
    ];

    const mediumRiskPatterns = [
      /rm\s+/,                     // 删除文件
      /mv\s+.*\/dev/,              // 移动到设备
      /echo\s+.*>\s*\//            // 写入系统文件
    ];

    for (const pattern of highRiskPatterns) {
      if (pattern.test(command)) {
        return "high";
      }
    }

    for (const pattern of mediumRiskPatterns) {
      if (pattern.test(command)) {
        return "medium";
      }
    }

    return "low";
  }
}
```

---

## 7. 运行时管理

### 7.1 端口管理

#### 7.1.1 端口可用性检查

```typescript
// src/infra/ports-probe.ts

/**
 * 尝试绑定到端口 (检查是否可用)
 */
export async function tryListenOnPort(params: {
  port: number;
  host?: string;
}): Promise<void> {
  return new Promise((resolve, reject) => {
    const server = net.createServer();

    server.once("error", (err) => {
      server.close();
      reject(err);
    });

    server.once("listening", () => {
      server.close(() => resolve());
    });

    server.listen(params.port, params.host ?? "127.0.0.1");
  });
}

/**
 * 确保端口可用
 */
export async function ensurePortAvailable(port: number): Promise<void> {
  try {
    await tryListenOnPort({ port });
  } catch (err) {
    if (isErrno(err) && err.code === "EADDRINUSE") {
      throw new PortInUseError(port, await describePortOwner(port));
    }
    throw err;
  }
}
```

#### 7.1.2 端口占用诊断

```typescript
// src/infra/ports-inspect.ts

interface PortUsage {
  port: number;
  protocol: "tcp" | "udp";
  address: string;
  state: "listen" | "time_wait" | "close_wait";
  pid?: number;
  processName?: string;
}

/**
 * 检查端口使用情况
 */
export async function inspectPortUsage(
  port: number
): Promise<PortUsage[]> {
  const listeners: PortUsage[] = [];

  // 使用 lsof 检查 (Unix-like)
  if (process.platform !== "win32") {
    try {
      const { stdout } = await execa("lsof", [
        "-i",
        `:${port}`,
        "-P",
        "-n",
      ]);

      listeners.push(...parseLsofOutput(stdout));
    } catch {
      // lsof 不可用，尝试 netstat
    }
  }

  // 使用 netstat 检查
  try {
    const { stdout } = await execa("netstat", [
      "-ano",
    ]);

    listeners.push(...parseNetstatOutput(stdout, port));
  } catch {
    // netstat 不可用
  }

  return listeners;
}
```

### 7.2 进程管理

#### 7.2.1 子进程执行

```typescript
// src/process/exec.ts

interface ExecOptions {
  cwd?: string;
  env?: Record<string, string>;
  timeout?: number;
  shell?: boolean;
  elevated?: boolean;
}

interface ExecResult {
  stdout: string;
  stderr: string;
  exitCode: number;
  signal?: string;
}

/**
 * 执行命令
 */
export async function runCommand(
  command: string,
  options: ExecOptions = {}
): Promise<ExecResult> {
  const controller = new AbortController();
  const timeoutId = options.timeout
    ? setTimeout(() => controller.abort(), options.timeout)
    : undefined;

  try {
    const result = await execa(command, {
      cwd: options.cwd,
      env: { ...process.env, ...options.env },
      shell: options.shell ?? true,
      signal: controller.signal,
      reject: false,
    });

    return {
      stdout: result.stdout,
      stderr: result.stderr,
      exitCode: result.exitCode,
      signal: result.signal,
    };
  } finally {
    if (timeoutId) {
      clearTimeout(timeoutId);
    }
  }
}

/**
 * 执行提升权限命令 (需要 sudo/admin)
 */
export async function runElevatedCommand(
  command: string
): Promise<ExecResult> {
  const sudoCommand = process.platform === "win32"
    ? `powershell -Command "Start-Process cmd -ArgumentList '/c ${command}' -Verb RunAs -Wait"`
    : `sudo -S ${command}`;

  return runCommand(sudoCommand, { shell: true });
}
```

---

## 8. 扩展开发指南

### 8.1 创建渠道插件

#### 8.1.1 最小渠道插件

```typescript
// my-channel-plugin/index.ts

import type { ChannelPlugin } from "openclaw/plugin-sdk/core";

const myChannel: ChannelPlugin = {
  id: "mychannel",
  meta: {
    id: "mychannel",
    label: "MyChannel",
    selectionLabel: "MyChannel (API)",
    docsPath: "/channels/mychannel",
    blurb: "我的渠道插件",
  },

  capabilities: {
    chatTypes: ["direct"],
  },

  config: {
    listAccountIds: (cfg) =>
      Object.keys(cfg.channels?.mychannel?.accounts ?? {}),

    resolveAccount: (cfg, accountId) =>
      cfg.channels?.mychannel?.accounts?.[accountId ?? "default"],
  },

  outbound: {
    deliveryMode: "direct",
    sendText: async ({ text, to, accountId, config }) => {
      // 实现发送逻辑
      const account = config.channels.mychannel.accounts[accountId];

      // 调用渠道 API
      const response = await fetch("https://api.mychannel.com/send", {
        method: "POST",
        headers: {
          "Authorization": `Bearer ${account.token}`,
          "Content-Type": "application/json",
        },
        body: JSON.stringify({
          to,
          text,
        }),
      });

      if (!response.ok) {
        throw new Error(`Failed to send: ${response.statusText}`);
      }

      const data = await response.json();
      return {
        ok: true,
        messageId: data.id,
      };
    },
  },
};

export default function(api) {
  api.registerChannel({ plugin: myChannel });
}
```

#### 8.1.2 清单文件

```json
// my-channel-plugin/openclaw.plugin.json

{
  "id": "mychannel",
  "name": "MyChannel Plugin",
  "version": "1.0.0",
  "description": "MyChannel messaging channel for OpenClaw",
  "configSchema": {
    "type": "object",
    "properties": {
      "channels": {
        "type": "object",
        "properties": {
          "mychannel": {
            "type": "object",
            "properties": {
              "accounts": {
                "type": "object",
                "additionalProperties": {
                  "type": "object",
                  "properties": {
                    "token": { "type": "string" },
                    "enabled": { "type": "boolean" }
                  }
                }
              }
            }
          }
        }
      }
    }
  },
  "uiHints": {
    "token": {
      "label": "API Token",
      "sensitive": true
    }
  }
}
```

### 8.2 创建工具插件

```typescript
// my-tool-plugin/index.ts

export default function(api) {
  api.registerTool({
    name: "weather_get",
    description: "Get current weather for a location",
    inputSchema: {
      type: "object",
      properties: {
        location: {
          type: "string",
          description: "City name or coordinates",
        },
      },
      required: ["location"],
    },
    async execute(params, ctx) {
      const response = await api.fetch(
        `https://api.weather.com/current?location=${encodeURIComponent(params.location)}`,
        {
          headers: {
            "Authorization": `Bearer ${ctx.config.services.weather.apiKey}`,
          },
        }
      );

      if (!response.ok) {
        return {
          error: `Failed to fetch weather: ${response.statusText}`,
        };
      }

      const data = await response.json();
      return {
        content: [
          {
            type: "text",
            text: `Weather in ${params.location}: ${data.condition}, ${data.temp}°C`,
          },
        ],
      };
    },
  });
}
```

### 8.3 创建认证提供者插件

```typescript
// my-auth-provider/index.ts

export default function(api) {
  api.registerProvider({
    id: "myprovider",
    label: "My AI Provider",
    auth: [
      {
        id: "api_key",
        label: "API Key",
        kind: "api_key",
        async run(ctx) {
          const apiKey = await ctx.prompter.text({
            message: "Enter your API key:",
            validate: (v) => v.length >= 10 ? undefined : "Key must be at least 10 characters",
          });

          return {
            profiles: [
              {
                profileId: "myprovider:default",
                credential: {
                  type: "api_key",
                  provider: "myprovider",
                  apiKey,
                },
              },
            ],
            defaultModel: "myprovider/standard",
          };
        },
      },
      {
        id: "oauth",
        label: "OAuth 2.0",
        kind: "oauth",
        async run(ctx) {
          const oauthConfig = {
            authorizationUrl: "https://auth.myprovider.com/oauth/authorize",
            tokenUrl: "https://auth.myprovider.com/oauth/token",
            clientId: ctx.config.oauth.clientId,
            scopes: ["chat", "models"],
          };

          const handlers = ctx.oauth.createVpsAwareHandlers(oauthConfig);
          const result = await ctx.oauth.runFlow(handlers);

          return {
            profiles: [
              {
                profileId: "myprovider:oauth",
                credential: {
                  type: "oauth",
                  provider: "myprovider",
                  access: result.accessToken,
                  refresh: result.refreshToken,
                  expires: result.expiresAt,
                },
              },
            ],
          };
        },
      },
    ],
  });
}
```

---

## 附录

### A. 类型定义汇总

```typescript
// 核心类型定义

type ThinkingLevel = "off" | "minimal" | "low" | "medium" | "high" | "xhigh";
type VerboseLevel = "off" | "on";

interface Config {
  gateway: GatewayConfig;
  models: ModelsConfig;
  channels: Record<string, ChannelConfig>;
  plugins: PluginsConfig;
  tools: ToolsConfig;
  logging: LoggingConfig;
}

interface GatewayConfig {
  bind: "loopback" | "lan" | "tailnet" | string;
  port: number;
  auth: AuthConfig;
}

interface ModelsConfig {
  default: string;
  providers: Record<string, ProviderConfig>;
  failover?: string[];
}

interface ProviderConfig {
  type: string;
  apiKey?: string;
  credentials?: OAuthCredentials;
}

interface ChannelConfig {
  accounts: Record<string, ChannelAccount>;
  enabled?: boolean;
  [key: string]: unknown;
}

interface PluginsConfig {
  enabled: boolean;
  allow?: string[];
  deny?: string[];
  slots: {
    memory?: string;
    contextEngine?: string;
  };
  entries: Record<string, PluginEntryConfig>;
}
```

### B. API 参考

#### B.1 Gateway RPC 方法

| 方法 | 描述 | 参数 |
|------|------|------|
| `connect` | 建立连接 | device, auth |
| `agent` | 执行 Agent | message, thinking, session |
| `send` | 发送消息 | channel, to, text |
| `chat.history` | 获取历史 | sessionKey, limit |
| `sessions.list` | 列出会话 | filter |
| `sessions.patch` | 更新会话 | sessionKey, patch |
| `config.get` | 获取配置 | path |
| `config.set` | 设置配置 | path, value |
| `health` | 健康检查 | - |
| `node.invoke` | 调用节点命令 | nodeId, command, params |

#### B.2 环境变

| 变量 | 描述 | 默认值 |
|------|------|--------|
| `OPENCLAW_CONFIG_PATH` | 配置文件路径 | `~/.openclaw/config.json` |
| `OPENCLAW_LOG_LEVEL` | 日志级别 | `info` |
| `OPENCLAW_GATEWAY_URL` | Gateway URL | `ws://127.0.0.1:18789` |
| `OPENCLAW_PLUGIN_CATALOG_PATHS` | 插件目录路径 | - |
| `OPENCLAW_SESSION_CACHE_TTL_MS` | 会话缓存 TTL | `45000` |

---

*文档版本：1.0*
*OpenClaw 版本：2026.3.9*
