# 第 4 章：网关配置详解

> 本章概述：深入讲解 OpenClaw 网关的配置方法，包括配置文件结构、网络设置、安全配置和远程访问。

## 学习目标

- 掌握配置文件的结构和编辑方法
- 理解网络绑定和端口配置
- 学会配置安全认证机制
- 了解远程访问的配置选项

## 前置条件

- 已完成 OpenClaw 的基础安装
- 了解 JSON/JSON5 基础语法

---

## 4.1 配置文件结构

### 4.1.1 配置文件位置

OpenClaw 的配置文件位于：

| 平台 | 路径 |
|------|------|
| macOS/Linux | `~/.openclaw/openclaw.json` |
| Windows (WSL2) | `~/.openclaw/openclaw.json` |
| Docker | 挂载卷中的 `/root/.openclaw/openclaw.json` |

<Note>
**配置文件格式**：
- 支持 JSON5 格式（允许注释和尾随逗号）
- 文件不存在时使用安全默认值
- 配置变更自动热重载（无需重启）
</Note>

### 4.1.2 最小可用配置

```json5
// ~/.openclaw/openclaw.json
{
  // Agent 工作区配置
  agents: {
    defaults: {
      workspace: "~/.openclaw/workspace"
    }
  },

  // WhatsApp 通道配置
  channels: {
    whatsapp: {
      allowFrom: ["+15555550123"]  // 允许的用户
    }
  }
}
```

### 4.1.3 配置文件层级

完整的配置结构包含以下主要部分：

```
配置根目录
├── gateway          # 网关设置（网络、认证、Tailscale）
├── agents           # Agent 配置（工作区、模型、工具）
├── channels         # 消息通道配置
├── session          # 会话管理配置
├── tools            # 工具策略配置
├── cron             # 定时任务配置
├── hooks            # Webhook 配置
├── browser          # 浏览器配置
└── plugins          # 插件配置
```

### 4.1.4 编辑配置文件的方法

**方法一：交互式向导**
```bash
# 完整设置向导
openclaw onboard

# 配置向导
openclaw configure
```

**方法二：CLI 命令**
```bash
# 获取配置值
openclaw config get agents.defaults.workspace

# 设置配置值
openclaw config set agents.defaults.heartbeat.every "2h"

# 删除配置值
openclaw config unset tools.web.search.apiKey
```

**方法三：控制面板**
```bash
# 打开浏览器中的配置界面
openclaw dashboard
```

**方法四：直接编辑**
```bash
# 使用编辑器直接修改
code ~/.openclaw/openclaw.json
```

---

## 4.2 网络设置

### 4.2.1 网络绑定配置

Gateway 默认绑定到本地回环地址，确保仅本地访问：

```json5
{
  gateway: {
    bind: "127.0.0.1",  // 绑定地址
    port: 18789         // 默认端口
  }
}
```

**绑定地址选项**：

| 地址 | 说明 | 安全级别 |
|------|------|----------|
| `127.0.0.1` | 本地回环（仅本机访问） | 高 |
| `0.0.0.0` | 所有网络接口（不推荐） | 低 |
| `192.168.x.x` | 局域网 IP | 中 |
| Tailscale IP | 尾网 IP | 高 |

<Warning>
**安全警告**：
- 不要将 `bind` 设置为 `0.0.0.0` 除非你完全了解风险
- 暴露到公网时，必须配置强认证（token 或 password）
- 优先使用 Tailscale 等安全网络进行远程访问
</Warning>

### 4.2.2 端口配置

默认端口为 `18789`，可以根据需要修改：

```json5
{
  gateway: {
    port: 18789  // 可以修改为其他端口
  }
}
```

**常用端口**：
- `18789` — 默认端口
- `8080`, `8888` — 常见替代端口
- `443` — 需要 root 权限（HTTPS）

### 4.2.3 WebSocket 协议

Gateway 使用 WebSocket 协议进行通信：

```
本地访问：ws://127.0.0.1:18789
远程访问：ws://<remote-ip>:18789
Tailscale: wss://<tailnet-address>:18789
```

**连接参数**：
```javascript
{
  type: "req",
  method: "connect",
  params: {
    auth: { token: "your-token" },
    client: "macos-app",
    version: "1.0.0"
  }
}
```

---

## 4.3 安全配置

### 4.3.1 认证模式

Gateway 支持多种认证模式：

```json5
{
  gateway: {
    auth: {
      // 认证模式
      mode: "token",  // token | password | tailscale | none

      // Token 认证（推荐）
      token: "your-secure-token-here",

      // 密码认证
      password: "your-secure-password",

      // Tailscale 特定配置
      allowTailscale: true,      // 允许 Tailscale 用户
      requireTailscale: false,   // 仅允许 Tailscale 用户
      useTailscaleUser: true     // 使用 Tailscale 用户身份
    }
  }
}
```

**认证模式对比**：

| 模式 | 安全性 | 适用场景 |
|------|--------|----------|
| `token` | 高 | 远程访问、自动化 |
| `password` | 高 | 远程访问、手动 |
| `tailscale` | 极高 | 尾网内访问 |
| `none` | 低 | 仅本地测试 |

### 4.3.2 Token 认证配置

**推荐方式：使用环境变量**
```json5
{
  gateway: {
    auth: {
      mode: "token"
      // token 从环境变量读取
    }
  }
}
```

设置环境变量：
```bash
# macOS/Linux
export OPENCLAW_GATEWAY_TOKEN="your-secure-token"

# 或添加到 ~/.bashrc, ~/.zshrc
echo 'export OPENCLAW_GATEWAY_TOKEN="your-secure-token"' >> ~/.bashrc

# Windows (PowerShell)
$env:OPENCLAW_GATEWAY_TOKEN="your-secure-token"
```

### 4.3.3 设备配对

设备配对用于管理客户端连接：

```json5
{
  gateway: {
    pairing: {
      // 配对策略
      policy: "auto-approve-local",  // auto | manual | auto-approve-local

      // 配对码长度
      codeLength: 6,

      // 配对码有效期（分钟）
      codeExpiryMinutes: 10
    }
  }
}
```

**配对命令**：
```bash
# 查看待批准的配对
openclaw pairing list

# 批准配对
openclaw pairing approve <channel> <code>

# 拒绝配对
openclaw pairing reject <channel> <code>

# 查看已配对设备
openclaw pairing approved
```

### 4.3.4 DM 访问控制

每个通道都可以配置 DM（私聊）访问策略：

```json5
{
  channels: {
    discord: {
      // DM 策略
      dmPolicy: "pairing",  // pairing | allowlist | open | disabled

      // 允许列表
      allowFrom: ["user-id-1", "user-id-2"],

      // 群组策略
      groupPolicy: "mention",  // mention | open | disabled
      groupAllowFrom: ["role-id-1"]
    }
  }
}
```

**DM 策略说明**：

| 策略 | 行为 |
|------|------|
| `pairing` | 未知发送者收到配对码（默认） |
| `allowlist` | 仅允许列表中的用户 |
| `open` | 允许所有 DM（需设置 `allowFrom: ["*"]`） |
| `disabled` | 拒绝所有 DM |

### 4.3.5 沙箱配置

启用 Docker 沙箱隔离非主会话：

```json5
{
  agents: {
    defaults: {
      sandbox: {
        // 沙箱模式
        mode: "non-main",  // off | non-main | all

        // 沙箱范围
        scope: "agent",    // session | agent | shared

        // 沙箱工作区根目录
        workspaceRoot: "~/.openclaw/sandboxes"
      }
    }
  }
}
```

**沙箱模式说明**：

| 模式 | 说明 |
|------|------|
| `off` | 禁用沙箱 |
| `non-main` | 非主会话使用沙箱（推荐） |
| `all` | 所有会话使用沙箱 |

**创建沙箱镜像**：
```bash
# 首次使用前运行脚本
./scripts/sandbox-setup.sh
```

---

## 4.4 远程访问

### 4.4.1 Tailscale 集成（推荐）

Tailscale 提供安全的远程访问方式：

```json5
{
  gateway: {
    // Tailscale 配置
    tailscale: {
      // 模式：off | serve | funnel
      mode: "serve",

      // 退出时重置
      resetOnExit: true,

      // 域名（可选）
      hostname: "my-openclaw"
    },

    // 绑定保持本地
    bind: "127.0.0.1",
    port: 18789
  }
}
```

**Tailscale 模式说明**：

| 模式 | 说明 | 访问范围 |
|------|------|----------|
| `off` | 禁用 Tailscale 自动化 | - |
| `serve` | HTTPS 尾网内访问 | 仅尾网设备 |
| `funnel` | 公网 HTTPS 访问 | 任何互联网设备 |

**访问地址**：
```
Serve 模式：https://my-openclaw.tailnet-name.ts.net
Funnel 模式：https://my-openclaw.tailnet-name.ts.net (公开)
```

<Note>
**Funnel 安全提示**：
- Funnel 模式会暴露到公网
- 必须配置 `auth.mode: "password"`
- 建议使用强密码并定期更换
</Note>

### 4.4.2 SSH 隧道

使用 SSH 隧道进行安全远程访问：

```bash
# 本地端口转发
ssh -N -L 18789:127.0.0.1:18789 user@gateway-host

# 然后本地访问
openclaw dashboard  # 访问 http://127.0.0.1:18789
```

### 4.4.3 远程配置示例

完整远程访问配置：

```json5
{
  gateway: {
    // 网络设置
    bind: "127.0.0.1",
    port: 18789,

    // 认证设置
    auth: {
      mode: "token",
      token: "${OPENCLAW_GATEWAY_TOKEN}"
    },

    // Tailscale 设置
    tailscale: {
      mode: "serve",
      resetOnExit: true,
      hostname: "my-claw"
    }
  },

  // 会话安全
  session: {
    dmScope: "per-channel-peer"
  },

  // 工具安全
  tools: {
    profile: "coding",
    deny: ["exec", "bash"]
  }
}
```

### 4.4.4 远程访问安全检查

运行安全检查：
```bash
# 检查安全配置
openclaw security audit

# 检查网关健康
openclaw health

# 查看当前连接
openclaw gateway connections
```

**检查清单**：
- [ ] 认证已启用（非 `none` 模式）
- [ ] 使用强 token/密码
- [ ] 绑定地址非 `0.0.0.0`
- [ ] Tailscale 或 SSH 隧道保护
- [ ] DM 策略设置为 `pairing` 或 `allowlist`
- [ ] 沙箱对非主会话启用

---

## 4.5 配置热重载

### 4.5.1 自动重载机制

Gateway 会监控配置文件变更并自动重载：

```json5
{
  gateway: {
    // 配置重载设置
    config: {
      watch: true,           // 启用文件监控
      debounceMs: 1000       // 防抖延迟（毫秒）
    }
  }
}
```

### 4.5.2 手动重载配置

```bash
# 重新加载配置
openclaw gateway reload

# 或通过信号
kill -SIGUSR1 <gateway-pid>
```

### 4.5.3 配置验证

```bash
# 验证配置
openclaw doctor

# 修复配置问题
openclaw doctor --fix
```

---

## 本章小结

- **配置文件**：位于 `~/.openclaw/openclaw.json`，支持 JSON5 格式
- **网络设置**：默认绑定 `127.0.0.1:18789`，远程访问需配置认证
- **安全配置**：支持 token/password/tailscale 认证，DM 配对机制
- **远程访问**：推荐 Tailscale Serve，备选 SSH 隧道
- **热重载**：配置变更自动生效，无需重启

## 延伸阅读

- [完整配置参考](https://docs.openclaw.ai/gateway/configuration-reference)
- [安全指南](https://docs.openclaw.ai/gateway/security)
- [Tailscale 集成](https://docs.openclaw.ai/gateway/tailscale)
- [第 5 章：消息通道配置](chapter-05.md)

---

*上一章：[第 3 章：核心概念](chapter-03.md) | 下一章：[第 5 章：消息通道配置](chapter-05.md)*
