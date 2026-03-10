# 飞书软件研发团队 Agent 配置教程

> 本教程详解如何基于 OpenClaw 搭建飞书消息通道的软件研发团队多 Agent 协作系统。
>
> **基于实际配置更新**：使用阿里云百炼模型 + 飞书通道

## 目录

1. [架构设计](#1-架构设计)
2. [前置准备](#2-前置准备)
3. [飞书应用创建](#3-飞书应用创建)
4. [OpenClaw 基础配置](#4-openclaw-基础配置)
5. [多 Agent 配置](#5-多-agent-配置)
6. [飞书通道配置](#6-飞书通道配置)
7. [AGENTS.md 配置](#7-agentsmd-配置)
8. [启动与测试](#8-启动与测试)
9. [常见问题](#9-常见问题)

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
# 配置目录已自动创建：~/.openclaw/
# 编辑配置文件：C:\Users\gmfen\.openclaw\openclaw.json
```

### 4.2 配置模型 Provider（阿里云百炼）

你的配置已使用阿里云百炼模型：

```json
{
  "models": {
    "providers": {
      "bailian": {
        "baseUrl": "https://coding.dashscope.aliyuncs.com/v1",
        "apiKey": "sk-sp-xxxxxx",
        "api": "openai-completions",
        "models": [
          { "id": "qwen3.5-plus", "name": "qwen3.5-plus" },
          { "id": "qwen3-coder-plus", "name": "qwen3-coder-plus" },
          { "id": "glm-5", "name": "glm-5" }
        ]
      }
    }
  }
}
```

### 4.3 配置 Gateway

```json
{
  "gateway": {
    "port": 18789,
    "mode": "local",
    "bind": "lan",
    "auth": {
      "mode": "token",
      "token": "your-token"
    }
  }
}
```

---

## 5. 多 Agent 配置

### 5.1 配置说明

软件研发团队需要 5 个 Agent 角色，在 `openclaw.json` 的 `agents.list` 中配置：

```json
{
  "agents": {
    "list": [
      {
        "id": "pm",
        "name": "PM Agent",
        "model": "bailian/qwen3.5-plus",
        "workspace": "C:\\Users\\gmfen\\.openclaw\\workspace-pm",
        "tools": { "profile": "full" }
      },
      {
        "id": "architect",
        "name": "Architect Agent",
        "model": "bailian/qwen3.5-plus",
        "workspace": "C:\\Users\\gmfen\\.openclaw\\workspace-architect"
      },
      {
        "id": "coder",
        "name": "Coder Agent",
        "model": "bailian/qwen3-coder-plus",
        "workspace": "C:\\Users\\gmfen\\.openclaw\\workspace-coder"
      },
      {
        "id": "reviewer",
        "name": "Reviewer Agent",
        "model": "bailian/qwen3.5-plus",
        "workspace": "C:\\Users\\gmfen\\.openclaw\\workspace-reviewer"
      },
      {
        "id": "tester",
        "name": "Tester Agent",
        "model": "bailian/qwen3-coder-next",
        "workspace": "C:\\Users\\gmfen\\.openclaw\\workspace-tester"
      }
    ]
  }
}
```

### 5.2 模型推荐

| Agent | 推荐模型 | 说明 |
|-------|----------|------|
| PM | qwen3.5-plus | 通用能力强，适合协调管理 |
| Architect | qwen3.5-plus | 架构设计需要强推理 |
| Coder | qwen3-coder-plus | 专用编程模型 |
| Reviewer | qwen3.5-plus | 综合能力强 |
| Tester | qwen3-coder-next | 快速生成测试 |

---

## 6. 飞书通道配置

### 6.1 配置飞书凭证

你的飞书配置已位于 `openclaw.json` 的 `channels.feishu`：

```json
{
  "channels": {
    "feishu": {
      "enabled": true,
      "appId": "cli_a924ca9fa4f85cc1",
      "appSecret": "Rbns4pWncPqP8BEQ4Ifyzbs4A1DGi4C3",
      "connectionMode": "websocket",
      "domain": "feishu",
      "groupPolicy": "open",
      "dmPolicy": "pairing",
      "requireMention": true
    }
  }
}
```

### 6.2 配置说明

| 参数 | 说明 | 当前值 |
|------|------|--------|
| `enabled` | 是否启用 | `true` |
| `appId` | 飞书应用 ID | `cli_xxx` |
| `appSecret` | 飞书应用密钥 | `Rbns4pW...` |
| `connectionMode` | 连接模式 | `websocket` |
| `groupPolicy` | 群组策略 | `open` (开放) |
| `dmPolicy` | DM 策略 | `pairing` (配对) |
| `requireMention` | 是否需要提及 | `true` |

### 6.3 修改访问控制

**重要**：建议将群组策略改为 `allowlist` 并配置允许的群组：

```json
{
  "channels": {
    "feishu": {
      "groupPolicy": "allowlist",
      "groups": [
        "oc_project_management",
        "oc_architecture_review",
        "oc_development_team"
      ]
    }
  }
}
```

---

## 7. AGENTS.md 配置

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

## 8. 启动与测试

### 8.1 验证配置

```bash
cd "C:/Users/gmfen/.openclaw"
openclaw config validate
```

### 8.2 启动 Gateway

```bash
# 启动 Gateway
openclaw gateway start

# 后台运行
openclaw gateway start --daemon

# 查看运行状态
openclaw gateway status
```

### 8.3 测试飞书连接

```bash
# 查看飞书通道状态
openclaw channels status feishu

# 测试连接
openclaw channels test feishu
```

### 8.4 配置飞书群组

在飞书中将应用添加到以下群组：
1. 项目管理群 → PM Agent
2. 架构设计群 → Architect Agent
3. 开发群 → Coder Agent
4. 代码审查群 → Reviewer Agent
5. 测试群 → Tester Agent

### 8.5 测试 Agent 协作

在飞书群中发送：
```
/pm 分析这个需求：实现用户登录功能
```

### 8.6 查看日志

```bash
# 实时查看日志
openclaw logs --follow

# 查看错误日志
openclaw logs --level error
```

---

## 9. 常见问题

### 9.1 飞书消息收不到

**检查清单**：
1. 确认应用已发布并添加到群组
2. 确认 WebSocket 连接正常
3. 检查 `dmPolicy` 和 `groupPolicy` 配置
4. 确认机器人在群组中

```bash
# 查看飞书通道状态
openclaw channels status feishu --verbose
```

### 9.2 Agent 路由错误

**诊断命令**：
```bash
# 查看 Agent 列表
openclaw agents list

# 查看会话状态
openclaw sessions --active

# 查看日志
openclaw logs | grep -E "routing|agent"
```

### 9.3 配置验证失败

```bash
# 验证配置
openclaw config validate

# 如果有错误，根据提示修复
```

### 9.4 工作区目录不存在

创建工作区目录并创建 AGENTS.md：

```bash
mkdir -p "C:/Users/gmfen/.openclaw/workspace-pm"
# 然后在该目录下创建 AGENTS.md 文件
```

---

## 附录：你的当前配置

### A.1 配置位置

- **配置文件**: `C:\Users\gmfen\.openclaw\openclaw.json`
- **工作区目录**: `C:\Users\gmfen\.openclaw\workspace-*`
- **飞书插件**: `@openclaw/feishu@2026.3.2`

### A.2 已配置的 Agent

| ID | 名称 | 模型 | 工作区 |
|----|------|------|--------|
| pm | PM Agent | qwen3.5-plus | workspace-pm |
| architect | Architect Agent | qwen3.5-plus | workspace-architect |
| coder | Coder Agent | qwen3-coder-plus | workspace-coder |
| reviewer | Reviewer Agent | qwen3.5-plus | workspace-reviewer |
| tester | Tester Agent | qwen3-coder-next | workspace-tester |

### A.3 飞书配置

- **App ID**: `cli_a924ca9fa4f85cc1`
- **连接模式**: WebSocket
- **群组策略**: Open (开放)
- **DM 策略**: Pairing (配对)
- **需要提及**: 是

---

## 下一步

- [第 8 章：多代理路由](../chapter-08.md)
- [第 9 章：工具系统](../chapter-09.md)
- [第 3 章：核心概念](../chapter-03.md)
