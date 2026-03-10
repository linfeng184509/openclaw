# 第 6 章：模型提供商配置

> 本章概述：讲解如何配置 LLM 模型提供商，包括 OpenAI、Anthropic、Google 等主流 provider，以及模型故障转移和认证管理。

## 学习目标

- 掌握主流模型提供商的配置方法
- 理解 API Key 轮换和 OAuth 认证
- 学会配置模型故障转移策略
- 了解会话绑定和冷却机制

## 前置条件

- 已拥有至少一个 LLM 提供商的 API Key 或 OAuth 账号

---

## 6.1 支持的模型提供商

### 6.1.1 内置提供商

OpenClaw 内置支持以下主流模型提供商：

| 提供商 | Provider ID | 认证方式 | 示例模型 |
|--------|-------------|----------|----------|
| OpenAI | `openai` | API Key | `openai/gpt-5.4` |
| Anthropic | `anthropic` | API Key / OAuth | `anthropic/claude-opus-4-6` |
| OpenAI Code | `openai-codex` | OAuth | `openai-codex/gpt-5.4` |
| Google Gemini | `google` | API Key | `google/gemini-3.1-pro` |
| Google Vertex | `google-vertex` | gcloud ADC | `google-vertex/gemini-pro` |
| OpenCode Zen | `opencode` | API Key | `opencode/claude-opus-4-6` |
| OpenRouter | `openrouter` | API Key | `openrouter/anthropic/claude` |
| Mistral | `mistral` | API Key | `mistral/mistral-large` |
| Groq | `groq` | API Key | `groq/llama-3.1` |
| Cerebras | `cerebras` | API Key | `cerebras/llama-3.3` |
| xAI | `xai` | API Key | `xai/grok-2` |
| Z.AI (GLM) | `zai` | API Key | `zai/glm-5` |
| Kilo Gateway | `kilocode` | API Key | `kilocode/anthropic/claude` |

### 6.1.2 自定义提供商

通过 `models.providers` 配置，可以添加自定义的 OpenAI 兼容提供商：

```json5
{
  models: {
    providers: {
      moonshot: {
        baseUrl: "https://api.moonshot.ai/v1",
        apiKey: "${MOONSHOT_API_KEY}",
        api: "openai-completions",
        models: [{ id: "kimi-k2.5", name: "Kimi K2.5" }]
      }
    }
  }
}
```

---

## 6.2 基础配置

### 6.2.1 最小配置

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "anthropic/claude-opus-4-6"
      }
    }
  }
}
```

### 6.2.2 配置环境变量

**方法 1：Shell 环境变量**
```bash
# OpenAI
export OPENAI_API_KEY="sk-..."

# Anthropic
export ANTHROPIC_API_KEY="sk-ant-..."

# Google Gemini
export GEMINI_API_KEY="..."

# OpenRouter
export OPENROUTER_API_KEY="..."
```

**方法 2：配置文件引用**
```json5
{
  agents: {
    defaults: {
      model: {
        primary: "openai/gpt-5.4"
      }
    }
  }
}
```

### 6.2.3 使用 Onboarding 配置

```bash
# OpenAI API Key
openclaw onboard --auth-choice openai-api-key

# Anthropic API Key 或 Token
openclaw onboard --auth-choice token

# Google Gemini
openclaw onboard --auth-choice gemini-api-key

# OpenAI Code (OAuth)
openclaw onboard --auth-choice openai-codex
```

---

## 6.3 API Key 轮换

### 6.3.1 环境变量轮换

OpenClaw 支持通过多种环境变量格式配置多个 API Key：

| 变量名 | 说明 |
|--------|------|
| `<PROVIDER>_API_KEY` | 主 API Key |
| `<PROVIDER>_API_KEY_1` | 第 1 个备用 Key |
| `<PROVIDER>_API_KEY_2` | 第 2 个备用 Key |
| `<PROVIDER>_API_KEYS` | 逗号或分号分隔的 Key 列表 |
| `OPENCLAW_LIVE_<PROVIDER>_KEY` | 最高优先级覆盖 |

**示例（多个 OpenAI Key）**：
```bash
export OPENAI_API_KEY="sk-key1"
export OPENAI_API_KEY_1="sk-key2"
export OPENAI_API_KEY_2="sk-key3"
# 或
export OPENAI_API_KEYS="sk-key1,sk-key2,sk-key3"
```

### 6.3.2 轮换行为

- **Key 选择顺序**：保留优先级并去重
- **429 限流时**：自动尝试下一个 Key
- **非限流错误**：立即失败，不轮换
- **所有 Key 失败**：返回最后一个错误

---

## 6.4 OAuth 认证

### 6.4.1 支持的 OAuth 提供商

| 提供商 | 说明 |
|--------|------|
| OpenAI Code | ChatGPT Plus 订阅 |
| Anthropic | Claude 订阅（setup-token） |
| Google Antigravity | Google 账号 |
| Google Gemini CLI | Google 账号 |
| Qwen Portal | 阿里账号 |

<Warning>
**OAuth 使用提示**：
- OAuth 集成可能是非官方的
- 某些提供商可能限制第三方工具使用
- 建议使用非关键账号进行 OAuth 认证
</Warning>

### 6.4.2 OAuth 登录流程

```bash
# OpenAI Code OAuth
openclaw models auth login --provider openai-codex

# Anthropic setup-token
claude setup-token
openclaw models auth paste-token --provider anthropic

# Google Antigravity
openclaw plugins enable google-antigravity-auth
openclaw models auth login --provider google-antigravity

# Qwen Portal
openclaw plugins enable qwen-portal-auth
openclaw models auth login --provider qwen-portal
```

### 6.4.3 OAuth 凭据存储

OAuth 凭据存储在：
```
~/.openclaw/agents/<agentId>/agent/auth-profiles.json
```

**凭据类型**：
- `type: "api_key"` — API Key 认证
- `type: "oauth"` — OAuth Token（含 refresh token）

---

## 6.5 模型故障转移

### 6.5.1 故障转移阶段

OpenClaw 分两阶段处理故障：

```
1. 认证 Profile 轮换（同一 Provider 内）
   ↓
2. 模型故障转移（切换到下一个模型）
```

### 6.5.2 配置故障转移

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "anthropic/claude-opus-4-6",
        fallbacks: [
          "openai/gpt-5.4",
          "google/gemini-3.1-pro",
          "openrouter/anthropic/claude"
        ]
      }
    }
  }
}
```

### 6.5.3 会话绑定

**会话绑定规则**：
- 每个会话固定使用一个认证 Profile
- 以下情况会重新选择：
  - 会话重置（`/new` 或 `/reset`）
  - 压缩完成
  - Profile 进入冷却期

**手动选择 Profile**：
```bash
/model anthropic/claude-opus-4-6@anthropic:user@gmail.com
```

### 6.5.4 冷却机制

**冷却时间（指数退避）**：

| 失败次数 | 冷却时间 |
|----------|----------|
| 第 1 次 | 1 分钟 |
| 第 2 次 | 5 分钟 |
| 第 3 次 | 25 分钟 |
| 第 4 次+ | 1 小时（上限） |

**存储位置**：
```json
{
  "usageStats": {
    "anthropic:user@gmail.com": {
      "lastUsed": 1736160000000,
      "cooldownUntil": 1736160600000,
      "errorCount": 2
    }
  }
}
```

### 6.5.5 计费失败禁用

计费/信用失败（如"信用不足"）会被禁用而非短期冷却：

```json
{
  "usageStats": {
    "provider:profile": {
      "disabledUntil": 1736178000000,
      "disabledReason": "billing"
    }
  }
}
```

**禁用退避**：
- 起始：5 小时
- 每次翻倍
- 上限：24 小时
- 24 小时无失败则重置

---

## 6.6 模型配置高级选项

### 6.6.1 模型目录和别名

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "anthropic/claude-sonnet-4-5"
      },
      models: {
        "anthropic/claude-sonnet-4-5": { alias: "Sonnet" },
        "anthropic/claude-opus-4-6": { alias: "Opus" },
        "openai/gpt-5.4": { alias: "GPT" }
      }
    }
  }
}
```

### 6.6.2 图像模型配置

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "anthropic/claude-opus-4-6"
      },
      imageModel: {
        primary: "openai/gpt-4o",
        fallbacks: ["google/gemini-3.1-flash"]
      }
    }
  }
}
```

### 6.6.3 自定义 Provider 配置

**Moonshot AI（Kimi）示例**：
```json5
{
  agents: {
    defaults: {
      model: {
        primary: "moonshot/kimi-k2.5"
      }
    }
  },
  models: {
    mode: "merge",
    providers: {
      moonshot: {
        baseUrl: "https://api.moonshot.ai/v1",
        apiKey: "${MOONSHOT_API_KEY}",
        api: "openai-completions",
        models: [
          { id: "kimi-k2.5", name: "Kimi K2.5" },
          { id: "kimi-k2-turbo-preview", name: "Kimi K2 Turbo" }
        ]
      }
    }
  }
}
```

**Volcano Engine（火山引擎）示例**：
```json5
{
  agents: {
    defaults: {
      model: {
        primary: "volcengine/doubao-seed-1-8-251228"
      }
    }
  }
}
```

---

## 6.7 模型选择最佳实践

### 6.7.1 推荐配置策略

**生产环境推荐**：
```json5
{
  agents: {
    defaults: {
      model: {
        // 主力模型（最强）
        primary: "anthropic/claude-opus-4-6",

        // 故障转移（次强）
        fallbacks: [
          "openai/gpt-5.4",
          "google/gemini-3.1-pro"
        ]
      }
    }
  }
}
```

**成本敏感场景**：
```json5
{
  agents: {
    defaults: {
      model: {
        primary: "anthropic/claude-sonnet-4-6",  // 中等
        fallbacks: [
          "openai/gpt-4o-mini",                   // 经济
          "groq/llama-3.1"                        // 快速便宜
        ]
      }
    }
  }
}
```

### 6.7.2 Provider 选择建议

| 场景 | 推荐 Provider |
|------|---------------|
| 代码生成 | OpenAI Code, Anthropic |
| 复杂推理 | Anthropic Claude Opus, OpenAI GPT-5 |
| 快速响应 | Groq, Cerebras |
| 成本敏感 | OpenRouter（免费模型）, Groq |
| 图像理解 | OpenAI GPT-4o, Google Gemini |
| 长上下文 | Anthropic（200K）, Google Gemini |

---

## 6.8 CLI 命令参考

### 6.8.1 模型管理

```bash
# 查看已配置模型
openclaw models list

# 查看模型状态（primary/fallbacks）
openclaw models status

# 设置主模型
openclaw models set anthropic/claude-opus-4-6

# 设置图像模型
openclaw models set-image openai/gpt-4o
```

### 6.8.2 别名管理

```bash
# 列出别名
openclaw models aliases list

# 添加别名
openclaw models aliases add Opus anthropic/claude-opus-4-6

# 移除别名
openclaw models aliases remove Opus
```

### 6.8.3 故障转移配置

```bash
# 列出 fallbacks
openclaw models fallbacks list

# 添加 fallback
openclaw models fallbacks add openai/gpt-5.4

# 移除 fallback
openclaw models fallbacks remove openai/gpt-5.4

# 清空 fallbacks
openclaw models fallbacks clear
```

---

## 本章小结

- **内置提供商**：支持 15+ 主流 LLM 提供商
- **API Key 轮换**：支持多 Key 配置，429 限流时自动切换
- **OAuth 认证**：支持 ChatGPT Plus、Claude 订阅等
- **故障转移**：两阶段处理（Profile 轮换 → 模型 fallback）
- **冷却机制**：指数退避，1 分钟 → 5 分钟 → 25 分钟 → 1 小时
- **会话绑定**：每个会话固定 Profile，重置/压缩后重新选择

## 延伸阅读

- [完整模型配置](https://docs.openclaw.ai/gateway/configuration#models)
- [Model Failover 详解](https://docs.openclaw.ai/concepts/model-failover)
- [Hugging Face 集成](https://docs.openclaw.ai/providers/huggingface)
- [第 7 章：安全与权限](chapter-07.md)

---

*上一章：[第 5 章：消息通道配置](chapter-05.md) | 下一章：[第 7 章：安全与权限](chapter-07.md)*
