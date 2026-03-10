# 飞书软件研发团队 Agent 配置教程

> 本教程详解如何基于 OpenClaw 搭建飞书消息通道的软件研发团队多 Agent 协作系统。

## 目录

1. [架构设计](#1-架构设计)
2. [前置准备](#2-前置准备)
3. [飞书应用创建](#3-飞书应用创建)
4. [OpenClaw 基础配置](#4-openclaw-基础配置)
5. [多 Agent 配置](#5-多-agent-配置)
6. [飞书通道配置](#6-飞书通道配置)
7. [Binding 路由配置](#7-binding-路由配置)
8. [Agent 间通信配置](#8-agent-间通信配置)
9. [AGENTS.md 配置](#9-agentsmd-配置)
10. [启动与测试](#10-启动与测试)
11. [常见问题](#11-常见问题)

---

## 1. 架构设计

### 1.1 团队 Agent 角色

```
┌─────────────────────────────────────────────────────────────────┐
│                        飞书网关                                  │
│                    (Feishu Gateway)                             │
└─────────────┬───────────────────────────────────────────────────┘
              │
    ┌─────────┼─────────┬─────────────┬─────────────┐
    │         │         │             │             │
    ▼         ▼         ▼             ▼             ▼
┌────────┐ ┌────────┐ ┌────────┐ ┌──────────┐ ┌──────────┐
│ PM     │ │ Architect│ │ Coder │ │ Reviewer │ │  Test   │
│ Agent  │ │ Agent  │ │ Agent  │ │ Agent    │ │  Agent   │
└───┬────┘ └───┬────┘ └───┬────┘ └────┬─────┘ └────┬─────┘
    │          │         │           │            │
    └──────────┴────┬────┴───────────┴────────────┘
                    │
         ┌──────────▼──────────┐
         │   Shared Memory     │
         │   (项目知识库)       │
         └─────────────────────┘
```

### 1.2 Agent 职责分工

| Agent | 职责 | 飞书群组 | 模型推荐 |
|-------|------|----------|----------|
| **PM Agent** | 需求分析、任务分解、进度跟踪 | 项目管理群 | claude-opus-4-6 |
| **Architect Agent** | 技术设计、架构评审、技术方案 | 架构设计群 | claude-opus-4-6 |
| **Coder Agent** | 代码实现、Code Review 修复 | 开发群 | claude-sonnet-4-6 |
| **Reviewer Agent** | 代码审查、安全检查、质量把控 | 代码审查群 | claude-opus-4-6 |
| **Test Agent** | 测试用例、自动化测试、Bug 验证 | 测试群 | claude-sonnet-4-6 |

---

## 2. 前置准备

### 2.1 软件要求

```bash
# 安装 OpenClaw
npm install -g @openclaw/core

# 验证安装
openclaw --version

# 确认 Node.js 版本（需要 18+）
node --version
```

### 2.2 账号要求

- 飞书企业管理员权限（用于创建自建应用）
- Anthropic API Key（或其他 LLM Provider）
- 稳定的网络环境

---

## 3. 飞书应用创建

### 3.1 创建飞书自建应用

1. 登录 [飞书开放平台](https://open.feishu.cn/)
2. 进入「企业后台」→「应用开发」→「自建应用」
3. 点击「创建应用」
   - 应用名称：`OpenClaw Gateway`
   - 应用图标：选择机器人图标
   - 应用类型：机器人

### 3.2 配置应用权限

在「权限管理」中添加以下权限：

```
- im:message：发送和接收消息
- im:chat：访问群聊信息
- im:contact：读取用户和部门信息
- contact:employee：读取员工信息（可选）
- drive:file：访问飞书文档（可选）
```

### 3.3 获取凭证

在「凭证信息」页面获取：
- **App ID** (`cli_xxxxxxxxxxxxx`)
- **App Secret** (`xxxxxxxxxxxxxxxx`)

### 3.4 配置事件订阅

1. 进入「事件订阅」页面
2. 设置订阅地址（Callback URL）：
   ```
   https://your-domain.com/openclaw/feishu
   ```
   > 注意：需要公网可访问的地址，可使用 ngrok 本地测试

3. 订阅以下事件：
   ```
   - im.message.receive_v1：收到消息
   - im.chat.member.modified_v1：群成员变更（可选）
   ```

4. 获取 **Verification Token** 和 **Encrypt Key**

### 3.5 发布应用

1. 进入「版本管理与发布」
2. 点击「发布」→「提交审核」（企业内部应用可跳过审核）
3. 在「可用范围」中添加要使用的飞书群组

---

## 4. OpenClaw 基础配置

### 4.1 初始化配置

```bash
# 创建配置目录
mkdir -p ~/.openclaw

# 初始化配置文件
openclaw init
```

### 4.2 配置 LLM Provider

编辑 `~/.openclaw/openclaw.json`：

```json5
{
  models: {
    providers: {
      anthropic: {
        apiKey: "sk-ant-xxxxxxxxxxxxxxxxxxxx",
        baseURL: "https://api.anthropic.com"
      },
      // 或使用其他 Provider
      openai: {
        apiKey: "sk-xxxxxxxxxxxxxxxxxxxx",
        baseURL: "https://api.openai.com/v1"
      }
    }
  },

  gateway: {
    bind: "0.0.0.0",
    port: 18789,
    auth: {
      tokens: ["your-gateway-token"]
    }
  }
}
```

---

## 5. 多 Agent 配置

### 5.1 创建 Agent 工作区

```bash
# 创建 PM Agent
openclaw agents add pm \
  --workspace ~/.openclaw/workspace-pm

# 创建 Architect Agent
openclaw agents add architect \
  --workspace ~/.openclaw/workspace-architect

# 创建 Coder Agent
openclaw agents add coder \
  --workspace ~/.openclaw/workspace-coder

# 创建 Reviewer Agent
openclaw agents add reviewer \
  --workspace ~/.openclaw/workspace-reviewer

# 创建 Test Agent
openclaw agents add test \
  --workspace ~/.openclaw/workspace-test

# 查看 Agent 列表
openclaw agents list
```

### 5.2 完整 agents.list 配置

编辑 `~/.openclaw/openclaw.json`：

```json5
{
  agents: {
    list: [
      {
        id: "pm",
        workspace: "~/.openclaw/workspace-pm",
        model: "anthropic/claude-opus-4-6",
        agentDir: "~/.openclaw/agents/pm/agent",
        tools: {
          profile: "project-management"
        }
      },
      {
        id: "architect",
        workspace: "~/.openclaw/workspace-architect",
        model: "anthropic/claude-opus-4-6",
        agentDir: "~/.openclaw/agents/architect/agent",
        tools: {
          profile: "architecture"
        }
      },
      {
        id: "coder",
        workspace: "~/.openclaw/workspace-coder",
        model: "anthropic/claude-sonnet-4-6",
        agentDir: "~/.openclaw/agents/coder/agent",
        tools: {
          profile: "coding"
        }
      },
      {
        id: "reviewer",
        workspace: "~/.openclaw/workspace-reviewer",
        model: "anthropic/claude-opus-4-6",
        agentDir: "~/.openclaw/agents/reviewer/agent",
        tools: {
          profile: "review"
        }
      },
      {
        id: "test",
        workspace: "~/.openclaw/workspace-test",
        model: "anthropic/claude-sonnet-4-6",
        agentDir: "~/.openclaw/agents/test/agent",
        tools: {
          profile: "testing"
        }
      }
    ]
  }
}
```

---

## 6. 飞书通道配置

### 6.1 基础通道配置

编辑 `~/.openclaw/openclaw.json`：

```json5
{
  channels: {
    feishu: {
      // 飞书应用凭证
      appId: "cli_xxxxxxxxxxxxx",
      appSecret: "xxxxxxxxxxxxxxxx",

      // 事件订阅配置
      verificationToken: "xxxxxxxxxxxxxxxx",
      encryptKey: "xxxxxxxxxxxxxxxx",

      // 消息接收地址（本地开发用 ngrok）
      webhookUrl: "https://your-ngrok-url/openclaw/feishu",

      // 访问控制
      dmPolicy: "allowlist",  // DM 访问策略
      allowFrom: [
        "ou_xxxxxxxx",  // PM 的飞书 ID
        "ou_yyyyyyyy"   // 架构师的飞书 ID
      ],

      // 群组策略
      groupPolicy: "allowlist",
      groups: [
        "oc_zzzzzzzzzz_project",    // 项目管理群
        "oc_zzzzzzzzzz_arch",       // 架构设计群
        "oc_zzzzzzzzzz_dev",        // 开发群
        "oc_zzzzzzzzzz_review",     // 代码审查群
        "oc_zzzzzzzzzz_test"        // 测试群
      ]
    }
  }
}
```

### 6.2 获取飞书 ID

```bash
# 查询用户飞书 ID
openclaw channels feishu lookup-user "@用户名"

# 查询群组飞书 ID
openclaw channels feishu lookup-chat "群组名称"
```

### 6.3 启动飞书通道

```bash
# 启动 Gateway
openclaw gateway start

# 查看通道状态
openclaw channels status feishu

# 测试连接
openclaw channels test feishu
```

---

## 7. Binding 路由配置

### 7.1 按群组路由

编辑 `~/.openclaw/openclaw.json`：

```json5
{
  bindings: [
    {
      // 项目管理群 → PM Agent
      agentId: "pm",
      match: {
        channel: "feishu",
        peer: { kind: "group", id: "oc_zzzzzzzzzz_project" }
      }
    },
    {
      // 架构设计群 → Architect Agent
      agentId: "architect",
      match: {
        channel: "feishu",
        peer: { kind: "group", id: "oc_zzzzzzzzzz_arch" }
      }
    },
    {
      // 开发群 → Coder Agent
      agentId: "coder",
      match: {
        channel: "feishu",
        peer: { kind: "group", id: "oc_zzzzzzzzzz_dev" }
      }
    },
    {
      // 代码审查群 → Reviewer Agent
      agentId: "reviewer",
      match: {
        channel: "feishu",
        peer: { kind: "group", id: "oc_zzzzzzzzzz_review" }
      }
    },
    {
      // 测试群 → Test Agent
      agentId: "test",
      match: {
        channel: "feishu",
        peer: { kind: "group", id: "oc_zzzzzzzzzz_test" }
      }
    }
  ]
}
```

### 7.2 按用户路由（DM）

```json5
{
  bindings: [
    {
      // PM 的 DM → PM Agent
      agentId: "pm",
      match: {
        channel: "feishu",
        peer: { kind: "direct", id: "ou_xxxxxxxx" }
      }
    },
    {
      // 架构师的 DM → Architect Agent
      agentId: "architect",
      match: {
        channel: "feishu",
        peer: { kind: "direct", id: "ou_yyyyyyyy" }
      }
    },
    {
      // 开发者的 DM → Coder Agent
      agentId: "coder",
      match: {
        channel: "feishu",
        peer: { kind: "direct", id: "ou_developer" }
      }
    }
  ]
}
```

### 7.3 按关键词路由

```json5
{
  bindings: [
    {
      // 包含"/review"的消息 → Reviewer Agent
      agentId: "reviewer",
      match: {
        channel: "feishu",
        content: { contains: ["/review", "代码审查"] }
      }
    },
    {
      // 包含"/test"的消息 → Test Agent
      agentId: "test",
      match: {
        channel: "feishu",
        content: { contains: ["/test", "测试用例"] }
      }
    }
  ]
}
```

---

## 8. Agent 间通信配置

### 8.1 agentToAgent 完整配置

```json5
{
  agents: {
    list: [
      {
        id: "pm",
        workspace: "~/.openclaw/workspace-pm",
        model: "anthropic/claude-opus-4-6",

        // PM Agent 可以调用其他 Agent
        agentToAgent: {
          allowedTargets: ["architect", "coder", "reviewer", "test"],
          mode: "async",
          timeout: 300,

          targets: {
            architect: {
              agentId: "architect",
              invokeMode: "sync",
              timeout: 600,
              inputMap: {
                requirement: "${currentRequirement}",
                context: "${projectContext}"
              }
            },
            coder: {
              agentId: "coder",
              invokeMode: "async",
              timeout: 900,
              callback: {
                notify: true,
                target: "pm",
                event: "code:complete"
              }
            },
            reviewer: {
              agentId: "reviewer",
              invokeMode: "sync",
              timeout: 300,
              inputMap: {
                diff: "${lastCommit.diff}",
                prUrl: "${pr.url}"
              }
            },
            test: {
              agentId: "test",
              invokeMode: "async",
              timeout: 600,
              inputMap: {
                feature: "${feature.description}",
                code: "${feature.code}"
              }
            }
          },

          // 工作流编排
          orchestration: {
            chain: ["architect", "coder", "reviewer", "test"]
          }
        }
      },
      {
        id: "coder",
        workspace: "~/.openclaw/workspace-coder",
        model: "anthropic/claude-sonnet-4-6",

        subAgent: {
          enabled: true,
          capabilities: [
            "code-generation",
            "refactoring",
            "bug-fix",
            "test-writing"
          ],
          languages: ["python", "javascript", "typescript", "go", "rust"],
          output: {
            format: "patch",
            includeExplanation: true
          }
        }
      },
      {
        id: "reviewer",
        workspace: "~/.openclaw/workspace-reviewer",
        model: "anthropic/claude-opus-4-6",

        subAgent: {
          enabled: true,
          capabilities: [
            "code-review",
            "security-audit",
            "performance-check"
          ],
          review: {
            checkSecurity: true,
            checkPerformance: true,
            checkStyle: true,
            suggestImprovements: true
          }
        }
      }
    ]
  }
}
```

---

## 9. AGENTS.md 配置

### 9.1 PM Agent 配置

创建 `~/.openclaw/workspace-pm/AGENTS.md`：

```markdown
# PM Agent 操作指令

## 角色定位

你是项目管理 Agent，负责：
- 需求分析和任务分解
- 进度跟踪和风险管理
- 协调各角色 Agent 协作

## 可调用的子代理

| 子代理 | 用途 | 命令 |
|--------|------|------|
| @architect | 技术方案设计 | `/delegate architect <需求>` |
| @coder | 代码实现 | `/delegate coder <任务>` |
| @reviewer | 代码审查 | `/delegate reviewer <PR>` |
| @test | 测试验证 | `/delegate test <功能>` |

## 工作流程

1. 收到需求后，先分析并创建任务列表
2. 委派架构师进行技术设计
3. 委派开发者实现功能
4. 委派审查者进行代码审查
5. 委派测试者进行验证

## 飞书命令

- `/pm status` - 查看项目进度
- `/pm delegate <agent> <task>` - 委派任务
- `/pm report` - 生成项目报告
```

### 9.2 Coder Agent 配置

创建 `~/.openclaw/workspace-coder/AGENTS.md`：

```markdown
# Coder Agent 操作指令

## 角色定位

你是软件开发 Agent，负责：
- 根据需求实现代码
- 编写单元测试
- 修复 Code Review 提出的问题

## 支持的语言

- Python
- JavaScript/TypeScript
- Go
- Rust
- Java

## 输出格式

代码输出使用 diff 格式：

```diff
+ def new_feature():
+     """实现新功能"""
+     return "feature"
```

## 开发规范

1. 先理解需求和上下文
2. 设计简洁的解决方案
3. 编写配套测试
4. 添加必要的注释

## 飞书命令

- `/code implement <task>` - 实现功能
- `/code fix <bug>` - 修复 Bug
- `/code refactor <file>` - 重构代码
- `/code test <file>` - 编写测试
```

### 9.3 Reviewer Agent 配置

创建 `~/.openclaw/workspace-reviewer/AGENTS.md`：

```markdown
# Reviewer Agent 操作指令

## 角色定位

你是代码审查 Agent，负责：
- 审查代码质量和安全性
- 提供改进建议
- 确保符合编码规范

## 审查检查项

- [ ] 代码逻辑正确
- [ ] 无安全漏洞
- [ ] 性能合理
- [ ] 遵循编码规范
- [ ] 有适当的测试
- [ ] 有必要的注释

## 审查输出格式

```markdown
## 审查结果

### 通过项
- 代码逻辑正确
- 测试覆盖充分

### 需要改进
1. 第 23 行：建议使用更明确的变量名
2. 第 45 行：添加错误处理

### 建议
- 考虑使用缓存优化性能
```

## 飞书命令

- `/review <pr>` - 审查 PR
- `/security-check <code>` - 安全检查
- `/perf-check <code>` - 性能检查
```

---

## 10. 启动与测试

### 10.1 启动 Gateway

```bash
# 验证配置
openclaw config validate

# 启动 Gateway
openclaw gateway start

# 后台运行
openclaw gateway start --daemon

# 查看运行状态
openclaw gateway status
```

### 10.2 测试飞书连接

```bash
# 发送测试消息到飞书群
openclaw channels feishu send \
  --chat "oc_zzzzzzzzzz_project" \
  --message "PM Agent 已上线，准备接收任务"

# 测试路由
openclaw gateway call bindings.match \
  --params '{"channel":"feishu","peer":{"kind":"group","id":"oc_zzzzzzzzzz_project"}}'
```

### 10.3 测试 Agent 协作

在飞书「项目管理群」发送：

```
/requirement 实现用户登录功能，支持邮箱和密码，使用 JWT token
```

预期流程：
1. PM Agent 接收需求并分析
2. PM Agent 委派 Architect Agent 设计技术方案
3. PM Agent 委派 Coder Agent 实现代码
4. Coder Agent 完成后通知 Reviewer Agent 审查
5. Reviewer Agent 审查通过后通知 Test Agent 编写测试

### 10.4 查看日志

```bash
# 实时查看日志
openclaw logs --follow

# 查看特定 Agent 日志
openclaw logs --agent pm --follow

# 查看错误日志
openclaw logs --level error
```

---

## 11. 常见问题

### 11.1 飞书消息收不到

**检查清单**：
1. 确认应用已发布并添加到群组
2. 确认事件订阅 URL 可访问
3. 确认 Verification Token 配置正确
4. 检查防火墙/网络设置

```bash
# 查看飞书通道状态
openclaw channels status feishu --verbose

# 测试 webhook
openclaw channels feishu test-webhook
```

### 11.2 Agent 路由错误

**诊断命令**：
```bash
# 查看 Binding 匹配
openclaw agents list --bindings

# 查看会话状态
openclaw sessions --active

# 查看路由日志
openclaw logs | grep -E "binding|routing"
```

### 11.3 agentToAgent 调用失败

**检查项**：
1. 确认目标 Agent 已配置 `subAgent.enabled: true`
2. 确认 `allowedTargets` 包含目标 Agent
3. 检查超时设置是否足够

```bash
# 测试子代理调用
openclaw agent test-invoke coder --input '{"task":"test"}'

# 查看调用历史
openclaw agent invoke-history --limit 10
```

### 11.4 飞书 ID 获取方法

```bash
# 方法 1：通过用户名查询
openclaw channels feishu lookup-user "@张三"

# 方法 2：通过群组名查询
openclaw channels feishu lookup-chat "项目管理群"

# 方法 3：查看消息日志获取 ID
openclaw logs --follow | grep -E "peer|chatId"
```

---

## 附录：完整配置示例

### A.1 完整 openclaw.json

```json5
{
  // LLM 配置
  models: {
    providers: {
      anthropic: {
        apiKey: "sk-ant-xxxxxxxxxxxxxxxxxxxx"
      }
    }
  },

  // Gateway 配置
  gateway: {
    bind: "0.0.0.0",
    port: 18789,
    auth: {
      tokens: ["your-gateway-token"]
    }
  },

  // 多 Agent 配置
  agents: {
    list: [
      {
        id: "pm",
        workspace: "~/.openclaw/workspace-pm",
        model: "anthropic/claude-opus-4-6"
      },
      {
        id: "architect",
        workspace: "~/.openclaw/workspace-architect",
        model: "anthropic/claude-opus-4-6"
      },
      {
        id: "coder",
        workspace: "~/.openclaw/workspace-coder",
        model: "anthropic/claude-sonnet-4-6"
      },
      {
        id: "reviewer",
        workspace: "~/.openclaw/workspace-reviewer",
        model: "anthropic/claude-opus-4-6"
      },
      {
        id: "test",
        workspace: "~/.openclaw/workspace-test",
        model: "anthropic/claude-sonnet-4-6"
      }
    ]
  },

  // 飞书通道配置
  channels: {
    feishu: {
      appId: "cli_xxxxxxxxxxxxx",
      appSecret: "xxxxxxxxxxxxxxxx",
      verificationToken: "xxxxxxxxxxxxxxxx",
      encryptKey: "xxxxxxxxxxxxxxxx",
      dmPolicy: "allowlist",
      allowFrom: ["ou_xxxxxxxx"],
      groupPolicy: "allowlist",
      groups: ["oc_zzzzzzzzzz_*"]
    }
  },

  // Binding 路由配置
  bindings: [
    {
      agentId: "pm",
      match: {
        channel: "feishu",
        peer: { kind: "group", id: "oc_zzzzzzzzzz_project" }
      }
    },
    {
      agentId: "coder",
      match: {
        channel: "feishu",
        peer: { kind: "group", id: "oc_zzzzzzzzzz_dev" }
      }
    },
    {
      agentId: "reviewer",
      match: {
        channel: "feishu",
        peer: { kind: "group", id: "oc_zzzzzzzzzz_review" }
      }
    }
  ]
}
```

---

## 下一步

- [多代理路由配置](../chapter-08.md)
- [工具系统配置](../chapter-09.md)
- [安全与权限](../chapter-07.md)
