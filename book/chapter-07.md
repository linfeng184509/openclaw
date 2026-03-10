# 第 7 章：安全与权限

> 本章概述：全面讲解 OpenClaw 的安全模型，包括 DM 配对机制、沙箱隔离、工具权限控制和 elevated 模式。

## 学习目标

- 理解 DM 访问控制和配对机制
- 掌握沙箱配置和隔离级别
- 学会配置工具权限策略
- 了解 elevated 模式的安全边界

## 前置条件

- 已完成基础 Gateway 配置
- 了解 Docker 基础概念（用于沙箱）

---

## 7.1 安全模型概述

### 7.1.1 默认安全边界

OpenClaw 的安全模型基于以下原则：

| 原则 | 说明 |
|------|------|
| **单用户信任** | 默认假设只有单一可信用户在操作 |
| **本地优先** | 数据存储在本地，不上传到云服务 |
| **最小权限** | 工具默认受限，按需开放 |
| **沙箱可选** | 非主会话可运行在 Docker 沙箱中 |

### 7.1.2 威胁模型

OpenClaw 主要防范：
- **意外操作**：模型执行危险命令
- **提示注入**：通过不可信输入执行的攻击
- **跨会话泄漏**：多用户场景下的隐私泄漏

<Warning>
**重要提醒**：
- 沙箱不是完美的安全边界
-  Elevated 模式会绕过沙箱
- 始终将不可信输入视为潜在威胁
</Warning>

---

## 7.2 DM 访问控制

### 7.2.1 DM 策略（dmPolicy）

所有通道共享相似的 DM 访问控制模式：

| 策略 | 行为 | 适用场景 |
|------|------|----------|
| `pairing` | 未知发送者收到配对码（默认） | 大多数场景 |
| `allowlist` | 仅允许列表中的用户 | 私人助理 |
| `open` | 允许所有（需 `allowFrom: ["*"]`） | 公共机器人 |
| `disabled` | 拒绝所有 DM | 仅群组使用 |

### 7.2.2 配对机制

**配对流程**：
```
1. 未知用户发送消息
2. Bot 回复配对码（如：ABCD1234）
3. 管理员收到配对请求
4. 管理员批准配对
5. 用户被添加到允许列表
```

**配对命令**：
```bash
# 查看待批准配对
openclaw pairing list [channel]

# 批准配对
openclaw pairing approve <channel> <CODE>

# 拒绝配对
openclaw pairing reject <channel> <CODE>

# 查看已批准设备
openclaw pairing approved --channel discord
```

### 7.2.3 安全 DM 模式（多用户场景）

**问题场景**：
```
Alice → Bot: "我的医疗预约是明天"
Bob   → Bot: "我们刚才在聊什么？"
结果  → Bot 可能基于 Alice 的上下文回答 Bob（隐私泄漏）
```

**解决方案：启用 DM 隔离**
```json5
{
  session: {
    dmScope: "per-channel-peer"  // 按渠道 + 发送者隔离
  }
}
```

**DM 作用域选项**：

| dmScope | 会话键格式 | 适用场景 |
|---------|-----------|----------|
| `main` | `agent:<agentId>:main` | 单用户（默认） |
| `per-peer` | `agent:<agentId>:dm:<peerId>` | 多用户，跨渠道共享 |
| `per-channel-peer` | `agent:<agentId>:<channel>:dm:<peerId>` | 多用户收件箱（推荐） |
| `per-account-channel-peer` | `agent:<agentId>:<channel>:<account>:dm:<peerId>` | 多账户 + 多用户 |

### 7.2.4 安全检查命令

```bash
# 安全审计
openclaw security audit

# 检查 DM 配置
openclaw config get session.dmScope
openclaw config get channels.discord.dmPolicy
```

---

## 7.3 沙箱隔离

### 7.3.1 沙箱概述

沙箱功能可以让工具在 Docker 容器中运行，减少潜在损害：

| 特性 | 说明 |
|------|------|
| **可选功能** | 默认禁用，需配置启用 |
| **工具隔离** | exec, read, write, edit 等工具在容器中运行 |
| **浏览器沙箱** | 可配置沙箱浏览器 |
| **非沙箱** | Gateway 进程本身、显式允许的工具 |

### 7.3.2 沙箱模式

`agents.defaults.sandbox.mode` 控制何时使用沙箱：

| 模式 | 说明 |
|------|------|
| `off` | 禁用沙箱 |
| `non-main` | 仅非主会话使用沙箱（推荐） |
| `all` | 所有会话使用沙箱 |

<Note>
**non-main 模式说明**：
- 基于 `session.mainKey`（默认 `"main"`）判断
- 群组/渠道会话被视为 non-main
- 直接聊天会话被视为主会话
</Note>

### 7.3.3 沙箱范围

`agents.defaults.sandbox.scope` 控制容器数量：

| 范围 | 说明 |
|------|------|
| `session` | 每个会话一个容器（默认） |
| `agent` | 每个代理一个容器 |
| `shared` | 所有沙箱会话共享一个容器 |

### 7.3.4 工作区访问

`agents.defaults.sandbox.workspaceAccess` 控制沙箱看到的内容：

| 访问级别 | 说明 |
|----------|------|
| `none`（默认） | 工具在 `~/.openclaw/sandboxes` 下运行 |
| `ro` | 只读挂载工作区到 `/agent` |
| `rw` | 读写挂载工作区到 `/workspace` |

### 7.3.5 配置示例

**最小沙箱配置**：
```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main",
        scope: "session",
        workspaceAccess: "none"
      }
    }
  }
}
```

**读写工作区**：
```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main",
        workspaceAccess: "rw"
      }
    }
  }
}
```

**自定义绑定挂载**：
```json5
{
  agents: {
    defaults: {
      sandbox: {
        docker: {
          binds: [
            "/home/user/source:/source:ro",
            "/var/data/myapp:/data:ro"
          ]
        }
      }
    }
  }
}
```

### 7.3.6 沙箱镜像设置

**构建基础沙箱镜像**：
```bash
# 基础镜像
scripts/sandbox-setup.sh

# 带常用工具的镜像（推荐）
scripts/sandbox-common-setup.sh

# 沙箱浏览器镜像
scripts/sandbox-browser-setup.sh
```

**镜像配置**：
```json5
{
  agents: {
    defaults: {
      sandbox: {
        docker: {
          image: "openclaw-sandbox-common:bookworm-slim",
          network: "openclaw-sandbox-network"
        }
      }
    }
  }
}
```

### 7.3.7 一次性设置命令

`setupCommand` 在容器创建后运行一次：

```json5
{
  agents: {
    defaults: {
      sandbox: {
        docker: {
          // 安装常用工具
          setupCommand: "apt-get update && apt-get install -y curl jq nodejs"
        }
      }
    }
  }
}
```

<Warning>
**setupCommand 注意事项**：
- 默认网络是 `"none"`（无出站），包安装会失败
- 需要设置 `network` 允许网络访问
- `readOnlyRoot: true` 会阻止写入
- 用户必须是 root 才能安装包
</Warning>

---

## 7.4 工具权限控制

### 7.4.1 工具配置文件

工具配置文件定义代理可以使用的工具集合：

| 配置文件 | 包含工具 | 适用场景 |
|----------|----------|----------|
| `minimal` | `session_status` | 最简对话 |
| `coding` | 文件系统、运行时、会话、记忆 | 编程任务 |
| `messaging` | 消息工具、会话列表 | 客服/通信 |
| `full` | 所有工具 | 完全访问 |

### 7.4.2 工具组速记

配置中可使用工具组：

| 组名 | 包含工具 |
|------|----------|
| `group:runtime` | exec, bash, process |
| `group:fs` | read, write, edit, apply_patch |
| `group:sessions` | sessions_list, sessions_history, sessions_send |
| `group:memory` | memory_search, memory_get |
| `group:web` | web_search, web_fetch |
| `group:ui` | browser, canvas |
| `group:messaging` | message |
| `group:nodes` | nodes |

### 7.4.3 工具策略配置

```json5
{
  tools: {
    // 基础配置
    profile: "coding",

    // 额外允许
    allow: ["browser", "slack"],

    // 明确拒绝
    deny: ["exec", "bash"],

    // 按提供商限制
    byProvider: {
      "google-antigravity": {
        profile: "minimal"
      }
    }
  }
}
```

### 7.4.4 沙箱工具策略

```json5
{
  agents: {
    defaults: {
      sandbox: {
        tools: {
          allow: ["bash", "process", "read", "write", "edit"],
          deny: ["browser", "canvas", "nodes", "cron"]
        }
      }
    }
  }
}
```

---

## 7.5 Elevated 模式

### 7.5.1 Elevated 模式概述

Elevated 模式允许 exec 工具在主机上运行，绕过沙箱：

| 特性 | 说明 |
|------|------|
| **显式授权** | 需要用户明确启用 |
| **绕过沙箱** | 直接在主机执行 |
| **会话级切换** | 每个会话独立控制 |
| **网关检查** | 需要配置允许列表 |

### 7.5.2 Elevated 配置

```json5
{
  tools: {
    // 全局 elevated 配置
    elevated: {
      enabled: true,
      allowlist: ["trusted-user-id"],
      requireApproval: true
    }
  },
  agents: {
    list: [{
      id: "main",
      tools: {
        // 代理级 elevated 覆盖
        elevated: {
          enabled: false  // 禁用 elevated
        }
      }
    }]
  }
}
```

### 7.5.3 Elevated 使用

**会话中启用**：
```
/elevated on
```

**会话中禁用**：
```
/elevated off
```

**检查状态**：
```
/elevated status
```

<Warning>
**Elevated 安全提醒**：
- Elevated 完全绕过沙箱隔离
- 只应信任的用户启用
- 生产环境建议禁用
- 沙箱关闭时 elevated 无额外效果
</Warning>

---

## 7.6 沙箱 vs 工具策略 vs Elevated

### 7.6.1 优先级顺序

```
1. 工具策略（Tools Policy）
   ↓ 允许的工具
2. Elevated 模式
   ↓ 在主机或沙箱中运行
3. 沙箱配置
   ↓ 文件系统/网络限制
```

### 7.6.2 决策树

```
工具调用
├── 工具在 deny 列表中？ → 拒绝
├── 工具不在 allow/profile 中？ → 拒绝
├── Elevated 启用且授权？ → 在主机运行
└── 否则
    ├── 沙箱模式为 "all"？ → 在沙箱运行
    ├── 沙箱模式为 "non-main" 且非主会话？ → 在沙箱运行
    └── 否则 → 在主机运行
```

### 7.6.3 诊断命令

```bash
# 检查沙箱配置
openclaw sandbox explain

# 安全审计
openclaw security audit

# 查看工具策略
openclaw config get tools
```

---

## 7.7 安全最佳实践

### 7.7.1 单用户场景

```json5
{
  // 基础安全配置
  session: {
    dmScope: "main"  // 单用户，连续性优先
  },
  agents: {
    defaults: {
      // 中等工具限制
      tools: {
        profile: "coding"
      }
    }
  }
}
```

### 7.7.2 多用户场景

```json5
{
  // 严格 DM 隔离
  session: {
    dmScope: "per-channel-peer"
  },
  agents: {
    defaults: {
      // 沙箱隔离
      sandbox: {
        mode: "non-main",
        workspaceAccess: "none"
      },
      // 消息工具限制
      tools: {
        profile: "messaging"
      }
    }
  }
}
```

### 7.7.3 公共机器人

```json5
{
  // 开放但受控
  channels: {
    telegram: {
      dmPolicy: "open",
      allowFrom: ["*"]
    }
  },
  agents: {
    defaults: {
      // 完全沙箱
      sandbox: {
        mode: "all",
        workspaceAccess: "none"
      },
      // 最简工具
      tools: {
        profile: "minimal"
      }
    }
  }
}
```

---

## 7.8 安全检查清单

### 7.8.1 启动前检查

```bash
# 1. 运行安全检查
openclaw security audit

# 2. 验证配置
openclaw doctor

# 3. 检查 DM 策略
openclaw config get channels.discord.dmPolicy

# 4. 检查沙箱配置
openclaw sandbox explain
```

### 7.8.2 持续监控

```bash
# 查看日志
openclaw logs --follow

# 检查健康状态
openclaw health

# 查看活跃会话
openclaw sessions --active 60
```

---

## 本章小结

- **DM 访问控制**：pairing/allowlist/open/disabled 四种策略
- **DM 隔离**：多用户场景使用 `dmScope: "per-channel-peer"`
- **沙箱隔离**：可选 Docker 容器，减少工具执行风险
- **工具策略**：通过 profile/allow/deny 控制可用工具
- **Elevated 模式**：显式授权在主机执行，绕过沙箱
- **安全检查**：使用 `openclaw security audit` 和 `openclaw doctor`

## 延伸阅读

- [沙箱详解](https://docs.openclaw.ai/gateway/sandboxing)
- [Elevated 模式](https://docs.openclaw.ai/tools/elevated)
- [安全指南](https://docs.openclaw.ai/gateway/security)
- [第 8 章：多代理路由](chapter-08.md)

---

*上一章：[第 6 章：模型提供商配置](chapter-06.md) | 下一章：[第 8 章：多代理路由](chapter-08.md)*
