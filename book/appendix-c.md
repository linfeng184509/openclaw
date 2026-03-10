# 附录 C：API 参考

> 本附录提供 OpenClaw WebSocket RPC 协议和 Gateway HTTP API 的完整参考。

## C.1 WebSocket RPC 协议

### C.1.1 连接 Gateway

```typescript
// 连接到 Gateway WebSocket
const ws = new WebSocket('ws://localhost:18789');

// 认证握手
ws.onopen = () => {
  ws.send(JSON.stringify({
    type: 'auth',
    token: 'your-token'  // 或 password: 'your-password'
  }));
};

// 监听响应
ws.onmessage = (event) => {
  const message = JSON.parse(event.data);
  console.log('Received:', message);
};
```

### C.1.2 RPC 消息格式

**请求格式**：

```typescript
{
  type: 'rpc',
  id: 'request-123',           // 唯一请求 ID
  method: 'gateway.status',    // RPC 方法
  params: {}                   // 参数对象
}
```

**响应格式**：

```typescript
{
  type: 'rpc',
  id: 'request-123',           // 对应请求 ID
  result: {...},               // 成功结果
  error: null                  // 或错误对象
}
```

**错误格式**：

```typescript
{
  code: -32000,                // 错误代码
  message: 'Error message',    // 错误消息
  data: {...}                  // 额外数据
}
```

### C.1.3 标准错误代码

| 代码 | 说明 |
|------|------|
| `-32700` | 解析错误（JSON 无效） |
| `-32600` | 无效请求 |
| `-32601` | 方法不存在 |
| `-32602` | 无效参数 |
| `-32603` | 内部错误 |
| `-32000` | 认证失败 |
| `-32001` | 权限不足 |

## C.2 Gateway RPC 方法

### C.2.1 Gateway 方法

**`gateway.status`**

获取 Gateway 状态。

```typescript
// 请求
{
  method: 'gateway.status',
  params: {}
}

// 响应
{
  runtime: 'running',
  version: '1.0.0',
  uptime: 123456,
  channels: {...},
  agents: {...}
}
```

**`gateway.health`**

健康检查。

```typescript
// 请求
{
  method: 'gateway.health',
  params: {}
}

// 响应
{
  status: 'healthy',
  checks: {
    channels: 'ok',
    agents: 'ok',
    database: 'ok'
  }
}
```

**`gateway.probe`**

探测 Gateway 可达性。

```typescript
// 请求
{
  method: 'gateway.probe',
  params: {}
}

// 响应
{
  reachable: true,
  latency: 5
}
```

### C.2.2 配置方法

**`config.get`**

获取配置。

```typescript
// 请求
{
  method: 'config.get',
  params: {
    path: 'gateway'  // 点分隔路径
  }
}

// 响应
{
  value: {...}
}
```

**`config.set`**

设置配置。

```typescript
// 请求
{
  method: 'config.set',
  params: {
    path: 'gateway.port',
    value: 18790
  }
}

// 响应
{
  success: true
}
```

**`config.apply`**

应用完整配置。

```typescript
// 请求
{
  method: 'config.apply',
  params: {
    config: {...},
    baseHash: 'abc123'  // 可选，用于乐观锁
  }
}
```

**`config.patch`**

部分更新配置。

```typescript
// 请求
{
  method: 'config.patch',
  params: {
    patch: {
      gateway: {
        port: 18790
      }
    }
  }
}
```

### C.2.3 Agent 方法

**`agent.run`**

运行 Agent。

```typescript
// 请求
{
  method: 'agent.run',
  params: {
    message: 'Hello',
    sessionKey: 'agent:main:main',
    thinking: 'medium',
    deliver: true
  }
}

// 响应
{
  ok: true,
  response: 'Hello! How can I help?',
  sessionKey: 'agent:main:main'
}
```

**`agent.stop`**

停止 Agent。

```typescript
// 请求
{
  method: 'agent.stop',
  params: {
    sessionKey: 'agent:main:main'
  }
}

// 响应
{
  ok: true
}
```

### C.2.4 会话方法

**`sessions.list`**

列出会话。

```typescript
// 请求
{
  method: 'sessions.list',
  params: {
    limit: 20,
    active: true
  }
}

// 响应
{
  sessions: [
    {
      key: 'agent:main:main',
      updatedAt: '2026-03-10T12:00:00Z',
      messageCount: 50
    }
  ]
}
```

**`sessions.describe`**

描述会话。

```typescript
// 请求
{
  method: 'sessions.describe',
  params: {
    sessionKey: 'agent:main:main'
  }
}

// 响应
{
  key: 'agent:main:main',
  createdAt: '2026-03-01T10:00:00Z',
  updatedAt: '2026-03-10T12:00:00Z',
  messageCount: 50,
  transcript: [...]
}
```

**`sessions.remove`**

删除会话。

```typescript
// 请求
{
  method: 'sessions.remove',
  params: {
    sessionKey: 'agent:main:main'
  }
}

// 响应
{
  ok: true
}
```

### C.2.5 Cron 方法

**`cron.list`**

列出 Cron 任务。

```typescript
// 请求
{
  method: 'cron.list',
  params: {}
}

// 响应
{
  jobs: [
    {
      id: 'job-123',
      name: 'Morning brief',
      enabled: true,
      schedule: {
        kind: 'cron',
        expr: '0 7 * * *',
        tz: 'Asia/Shanghai'
      }
    }
  ]
}
```

**`cron.add`**

添加 Cron 任务。

```typescript
// 请求
{
  method: 'cron.add',
  params: {
    name: 'Morning brief',
    schedule: {
      kind: 'cron',
      expr: '0 7 * * *',
      tz: 'Asia/Shanghai'
    },
    sessionTarget: 'isolated',
    payload: {
      kind: 'agentTurn',
      message: 'Summarize overnight updates.'
    },
    delivery: {
      mode: 'announce',
      channel: 'whatsapp',
      to: '+8613800138000'
    }
  }
}

// 响应
{
  ok: true,
  jobId: 'job-123'
}
```

**`cron.edit`**

编辑 Cron 任务。

```typescript
// 请求
{
  method: 'cron.edit',
  params: {
    jobId: 'job-123',
    patch: {
      enabled: false
    }
  }
}
```

**`cron.run`**

运行 Cron 任务。

```typescript
// 请求
{
  method: 'cron.run',
  params: {
    jobId: 'job-123',
    mode: 'force'  // 或 'due'
  }
}
```

**`cron.remove`**

删除 Cron 任务。

```typescript
// 请求
{
  method: 'cron.remove',
  params: {
    jobId: 'job-123'
  }
}
```

### C.2.6 节点方法

**`node.list`**

列出节点。

```typescript
// 请求
{
  method: 'node.list',
  params: {}
}

// 响应
{
  nodes: [
    {
      id: 'node-123',
      name: 'macOS Node',
      role: 'node',
      connected: true,
      lastSeen: '2026-03-10T12:00:00Z'
    }
  ]
}
```

**`node.describe`**

描述节点。

```typescript
// 请求
{
  method: 'node.describe',
  params: {
    nodeId: 'node-123'
  }
}

// 响应
{
  id: 'node-123',
  name: 'macOS Node',
  capabilities: ['canvas', 'camera', 'screen'],
  permissions: {...}
}
```

**`node.invoke`**

调用节点命令。

```typescript
// 请求
{
  method: 'node.invoke',
  params: {
    nodeId: 'node-123',
    command: 'canvas.navigate',
    params: {
      url: 'http://localhost:18789/__openclaw__/canvas/'
    }
  }
}

// 响应
{
  ok: true,
  result: {...}
}
```

### C.2.7 设备方法

**`device.list`**

列出设备请求。

```typescript
// 请求
{
  method: 'device.list',
  params: {}
}

// 响应
{
  requests: [
    {
      id: 'req-123',
      deviceName: 'iPhone',
      channel: 'whatsapp',
      timestamp: '2026-03-10T12:00:00Z'
    }
  ]
}
```

**`device.approve`**

批准设备。

```typescript
// 请求
{
  method: 'device.approve',
  params: {
    requestId: 'req-123'
  }
}

// 响应
{
  ok: true
}
```

**`device.reject`**

拒绝设备。

```typescript
// 请求
{
  method: 'device.reject',
  params: {
    requestId: 'req-123'
  }
}

// 响应
{
  ok: true
}
```

### C.2.8 浏览器方法

**`browser.status`**

浏览器状态。

```typescript
// 请求
{
  method: 'browser.status',
  params: {}
}

// 响应
{
  running: true,
  profile: 'default',
  tabs: [...]
}
```

**`browser.navigate`**

导航到 URL。

```typescript
// 请求
{
  method: 'browser.navigate',
  params: {
    url: 'https://example.com'
  }
}

// 响应
{
  ok: true
}
```

## C.3 事件订阅

### C.3.1 订阅事件

```typescript
// 订阅事件
ws.send(JSON.stringify({
  type: 'subscribe',
  channel: 'agent:main:main'
}));

// 接收事件
ws.onmessage = (event) => {
  const message = JSON.parse(event.data);
  if (message.type === 'event') {
    console.log('Event:', message.event);
  }
};
```

### C.3.2 事件类型

**Agent 事件**：

```typescript
{
  type: 'event',
  event: {
    type: 'agent:turn:start',
    sessionKey: 'agent:main:main',
    turnId: 'turn-123'
  }
}
```

**消息事件**：

```typescript
{
  type: 'event',
  event: {
    type: 'message:sent',
    sessionKey: 'agent:main:main',
    content: 'Hello!',
    channel: 'whatsapp',
    to: '+8613800138000'
  }
}
```

**Cron 事件**：

```typescript
{
  type: 'event',
  event: {
    type: 'cron:finished',
    jobId: 'job-123',
    result: {...}
  }
}
```

## C.4 HTTP API

### C.4.1 健康检查端点

**GET /healthz**

存活探针（无需认证）。

```bash
curl http://localhost:18789/healthz
```

**响应**：

```json
{
  "status": "ok"
}
```

**GET /readyz**

就绪探针（无需认证）。

```bash
curl http://localhost:18789/readyz
```

**响应**：

```json
{
  "status": "ok",
  "checks": {
    "channels": true,
    "agents": true
  }
}
```

**GET /health**

完整健康检查（需要认证）。

```bash
curl -H "Authorization: Bearer your-token" \
  http://localhost:18789/health
```

### C.4.2 Dashboard

**GET /**

Web Dashboard（需要浏览器认证）。

```
http://localhost:18789/
```

### C.4.3 Canvas 端点

**GET /__openclaw__/canvas/**

Canvas 主机。

```
http://localhost:18789/__openclaw__/canvas/
```

**GET /__openclaw__/a2ui/**

A2UI 自动化界面。

```
http://localhost:18789/__openclaw__/a2ui/
```

## C.5 认证方式

### C.5.1 Token 认证

```typescript
ws.send(JSON.stringify({
  type: 'auth',
  token: 'your-token'
}));
```

**HTTP Header**：

```
Authorization: Bearer your-token
```

### C.5.2 密码认证

```typescript
ws.send(JSON.stringify({
  type: 'auth',
  password: 'your-password'
}));
```

### C.5.3 设备认证

设备通过配对码认证：

```typescript
ws.send(JSON.stringify({
  type: 'auth',
  device: {
    name: 'iPhone',
    role: 'node',
    pairingCode: 'ABC123'
  }
}));
```

### C.5.4 Tailscale 认证

通过 Tailscale 身份头认证：

```
Authorization: Bearer <Tailscale 身份头>
```

## C.6 速率限制

### C.6.1 限制策略

| 端点 | 限制 |
|------|------|
| RPC 调用 | 60 次/分钟 |
| 健康检查 | 无限 |
| 文件上传 | 10MB/请求 |

### C.6.2 超限响应

```json
{
  "error": {
    "code": -32002,
    "message": "Rate limit exceeded",
    "data": {
      "retryAfter": 60
    }
  }
}
```

## C.7 错误处理

### C.7.1 错误格式

```typescript
{
  type: 'rpc',
  id: 'request-123',
  error: {
    code: -32000,
    message: 'Authentication failed',
    data: {
      reason: 'invalid_token'
    }
  }
}
```

### C.7.2 常见错误

| 错误 | 代码 | 说明 |
|------|------|------|
| 解析错误 | -32700 | JSON 格式无效 |
| 无效请求 | -32600 | 缺少必需字段 |
| 方法不存在 | -32601 | RPC 方法不存在 |
| 无效参数 | -32602 | 参数格式错误 |
| 内部错误 | -32603 | 服务器内部错误 |
| 认证失败 | -32000 | token/password 无效 |
| 权限不足 | -32001 | 无权执行操作 |
| 速率限制 | -32002 | 请求过于频繁 |
| 会话过期 | -32003 | 会话已过期 |
| 资源不存在 | -32004 | 请求的资源不存在 |

---

*返回：[附录 B：配置 Schema](appendix-b.md) | 下一章：[附录 D：术语表](appendix-d.md)*
