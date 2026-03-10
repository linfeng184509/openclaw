# 附录 B：配置 Schema

> 本附录提供 OpenClaw 配置文件的完整 Schema 参考，包括所有配置项、类型和默认值。

## B.1 配置文件位置

```bash
# 默认位置
~/.openclaw/openclaw.json

# 通过环境变量覆盖
export OPENCLAW_CONFIG_DIR=/path/to/config
```

## B.2 根级别配置

```json5
{
  // Gateway 配置
  "gateway": {...},

  // Agent 配置
  "agents": {...},

  // 通道配置
  "channels": {...},

  // 工具配置
  "tools": {...},

  // 模型配置
  "models": {...},

  // 消息配置
  "messages": {...},

  // Cron 调度
  "cron": {...},

  // Hooks 配置
  "hooks": {...},

  // 浏览器配置
  "browser": {...},

  // 沙箱配置
  "sandbox": {...},

  // 插件配置
  "plugins": {...},

  // 日志配置
  "logging": {...},

  // 远程配置
  "gateway.remote": {...}
}
```

## B.3 Gateway 配置

### B.3.1 基础配置

```json5
{
  gateway: {
    // Gateway 模式
    mode: "local",                    // "local" | "remote"

    // 网络绑定
    bind: "loopback",                 // "loopback" | "lan" | "tailnet" | "auto" | "custom"
    port: 18789,                      // 端口号
    host: "0.0.0.0",                  // 自定义主机

    // 认证配置
    auth: {
      mode: "token",                  // "token" | "password" | "device"
      token: "your-token",            // 认证 token
      password: "your-password",      // 认证密码
      allowTailscale: false,          // 允许 Tailscale 身份认证
      deviceAuth: true                // 设备认证
    },

    // Tailscale 配置
    tailscale: {
      mode: "off",                    // "off" | "serve" | "funnel"
      resetOnExit: false              // 退出时重置
    }
  }
}
```

### B.3.2 远程配置

```json5
{
  gateway: {
    remote: {
      url: "wss://gateway.example.com",    // 远程 Gateway URL
      token: "your-token",                 // 远程认证 token
      tlsFingerprint: "SHA256:..."         // TLS 指纹（可选）
    }
  }
}
```

## B.4 Agent 配置

### B.4.1 默认配置

```json5
{
  agents: {
    defaults: {
      // 模型配置
      model: {
        primary: "anthropic/claude-opus-4-6",
        fallbacks: ["anthropic/claude-sonnet-4-6"],
        imagePrimary: "anthropic/claude-opus-4-6",
        imageFallbacks: []
      },

      // 会话配置
      session: {
        compact: {
          threshold: 100,              // 压缩阈值（消息数）
          target: 50                   // 压缩后目标消息数
        },
        idleExpiry: "24h"              // 空闲过期时间
      },

      // Heartbeat 配置
      heartbeat: {
        every: "30m",                  // 心跳间隔
        target: "none",                // 投递目标
        prompt: "Read HEARTBEAT.md...",
        activeHours: {
          start: "08:00",
          end: "22:00",
          timezone: "Asia/Shanghai"
        }
      },

      // 工具配置
      tools: {
        alsoAllow: [],                 // 额外允许的工具
        deny: [],                      // 拒绝的工具
        elevated: ["exec"]             // 提升权限的工具
      },

      // 沙箱配置
      sandbox: {
        mode: "non-main",              // "off" | "non-main" | "all"
        scope: "agent"                 // "session" | "agent" | "shared"
      },

      // 执行审批
      execApprovals: {
        security: "allowlist",         // "deny" | "allowlist" | "full"
        ask: "on-miss"                 // "on-miss" | "always" | "off"
      }
    }
  }
}
```

### B.4.2 Agent 列表

```json5
{
  agents: {
    list: [
      {
        id: "main",                    // Agent ID
        default: true,                 // 是否为默认 Agent
        workspace: "~/.openclaw/workspace/main",
        model: {
          primary: "anthropic/claude-opus-4-6"
        },
        heartbeat: {
          every: "1h",
          target: "whatsapp"
        }
      },
      {
        id: "ops",
        workspace: "~/.openclaw/workspace/ops",
        heartbeat: {
          every: "30m",
          target: "telegram",
          to: "+1234567890"
        }
      }
    ]
  }
}
```

## B.5 通道配置

### B.5.1 通用配置

```json5
{
  channels: {
    // 默认配置
    defaults: {
      dmPolicy: "allowlist",           // "open" | "allowlist" | "deny"
      groupPolicy: "allowlist",
      allowFrom: ["+8613800138000"],   // 允许的发送者
      requireMention: false,           // 群组需要提及
      historyLimit: 50                 // 历史消息限制
    },

    // WhatsApp 配置
    whatsapp: {
      accounts: {
        default: {
          phoneNumber: "+8613800138000",
          // Baileys 配置
        }
      },
      allowFrom: [],
      groups: {
        "*": {
          requireMention: true
        }
      }
    },

    // Telegram 配置
    telegram: {
      accounts: {
        default: {
          botToken: "YOUR_BOT_TOKEN"
        }
      },
      allowFrom: [],
      groups: {
        "*": {
          requireMention: false
        }
      },
      customCommands: [
        {
          command: "/start",
          response: "Hello!"
        }
      ]
    },

    // Discord 配置
    discord: {
      accounts: {
        default: {
          botToken: "YOUR_BOT_TOKEN"
        }
      },
      guilds: {
        "*": {
          requireMention: true
        }
      }
    },

    // Slack 配置
    slack: {
      accounts: {
        default: {
          botToken: "xoxb-..."
        }
      }
    },

    // Signal 配置
    signal: {
      accounts: {
        default: {
          phoneNumber: "+8613800138000"
        }
      }
    },

    // iMessage 配置
    imessage: {
      enabled: true,
      handle: "your-handle"
    }
  }
}
```

## B.6 模型配置

### B.6.1 模型提供商

```json5
{
  models: {
    // 提供商配置
    providers: {
      anthropic: {
        apiKey: "sk-ant-...",
        baseURL: "https://api.anthropic.com"
      },
      openai: {
        apiKey: "sk-...",
        baseURL: "https://api.openai.com/v1"
      },
      gemini: {
        apiKey: "AIza...",
        baseURL: "https://generativelanguage.googleapis.com"
      },
      ollama: {
        baseURL: "http://localhost:11434"
      }
    },

    // 模型别名
    aliases: {
      "opus": "anthropic/claude-opus-4-6",
      "sonnet": "anthropic/claude-sonnet-4-6",
      "gpt-4": "openai/gpt-4o"
    },

    // 思考级别配置
    thinking: {
      enabled: true,
      defaultLevel: "medium"           // "off" | "minimal" | "low" | "medium" | "high" | "xhigh"
    }
  }
}
```

## B.7 工具配置

### B.7.1 工具控制

```json5
{
  tools: {
    // 浏览器工具
    browser: {
      enabled: true,
      executablePath: "/usr/bin/google-chrome",
      cdpPort: 9222,
      headless: true
    },

    // 媒体工具
    media: {
      audio: {
        models: ["openai/whisper-1"],
        tts: {
          provider: "elevenlabs",
          voice: "default"
        }
      }
    },

    // 沙箱工具
    sandbox: {
      enabled: true,
      docker: {
        image: "openclaw-sandbox:bookworm-slim",
        network: "none",
        user: "1000:1000",
        readOnlyRoot: true
      }
    },

    // Agent 到 Agent 工具
    agentToAgent: {
      enabled: true
    }
  }
}
```

### B.7.2 执行审批配置

```json5
{
  tools: {
    execApprovals: {
      defaults: {
        security: "allowlist",
        ask: "on-miss"
      },
      agents: {
        main: {
          security: "allowlist",
          allowlist: [
            { pattern: "/opt/homebrew/bin/rg" },
            { pattern: "/usr/bin/git" }
          ]
        }
      }
    }
  }
}
```

## B.8 Cron 配置

```json5
{
  cron: {
    enabled: true,
    store: "~/.openclaw/cron/jobs.json",
    maxConcurrentRuns: 1,

    // 重试策略
    retry: {
      maxAttempts: 3,
      backoffMs: [60000, 120000, 300000],
      retryOn: ["rate_limit", "overloaded", "network", "server_error"]
    },

    // Webhook 配置（废弃的回退）
    webhook: "https://example.invalid/legacy",
    webhookToken: "your-bearer-token",

    // 会话保留
    sessionRetention: "24h",

    // 运行日志
    runLog: {
      maxBytes: "2mb",
      keepLines: 2000
    }
  }
}
```

## B.9 Hooks 配置

```json5
{
  hooks: {
    internal: {
      enabled: true,
      entries: {
        "session-memory": { enabled: true },
        "command-logger": { enabled: false },
        "bootstrap-extra-files": {
          enabled: true,
          paths: ["packages/*/AGENTS.md"]
        },
        "boot-md": { enabled: true }
      },
      load: {
        extraDirs: ["/path/to/more/hooks"]
      }
    }
  }
}
```

## B.10 浏览器配置

```json5
{
  browser: {
    enabled: true,
    executablePath: "/usr/bin/google-chrome",
    cdpPort: 9222,
    headless: true,
    userDataDir: "~/.openclaw/browser-profiles/default",

    // SSRF 策略
    ssrfPolicy: {
      dangerouslyAllowPrivateNetwork: false
    },

    // 配置文件
    profiles: [
      {
        name: "default",
        color: "#FF5A2D"
      }
    ]
  }
}
```

## B.11 沙箱配置

```json5
{
  sandbox: {
    enabled: true,
    mode: "non-main",
    scope: "agent",
    workspaceAccess: "none",

    docker: {
      image: "openclaw-sandbox:bookworm-slim",
      network: "none",
      user: "1000:1000",
      readOnlyRoot: true,
      setupCommand: "apt-get update && apt-get install -y git curl jq",
      pidsLimit: 256,
      memory: "1g",
      cpus: 1
    }
  }
}
```

## B.12 插件配置

```json5
{
  plugins: {
    entries: {
      "my-plugin": {
        enabled: true,
        config: {
          apiKey: "..."
        }
      }
    },
    load: {
      paths: ["~/.openclaw/plugins"]
    }
  }
}
```

## B.13 日志配置

```json5
{
  logging: {
    level: "info",                     // "debug" | "info" | "warn" | "error"
    format: "json",                    // "json" | "text"
    output: {
      file: "~/.openclaw/logs/gateway.log",
      console: true
    },
    rotation: {
      maxFileSize: "10mb",
      maxFiles: 5
    }
  }
}
```

## B.14 消息配置

```json5
{
  messages: {
    // 群组聊天配置
    groupChat: {
      requireMention: false,
      mentionPatterns: ["@bot", "@Bot"],
      historyLimit: 50
    },

    // 队列配置
    queue: {
      maxConcurrency: 1,
      timeout: "5m"
    },

    // 确认配置
    ack: {
      reaction: true
    }
  }
}
```

## B.15 完整配置示例

```json5
{
  gateway: {
    mode: "local",
    bind: "loopback",
    port: 18789,
    auth: {
      mode: "token",
      token: "${GATEWAY_TOKEN}"
    }
  },

  agents: {
    defaults: {
      model: {
        primary: "anthropic/claude-opus-4-6",
        fallbacks: ["anthropic/claude-sonnet-4-6"]
      },
      heartbeat: {
        every: "30m",
        target: "last"
      },
      session: {
        compact: {
          threshold: 100,
          target: 50
        }
      }
    },
    list: [
      {
        id: "main",
        default: true,
        workspace: "~/.openclaw/workspace"
      }
    ]
  },

  channels: {
    defaults: {
      dmPolicy: "allowlist",
      allowFrom: ["+8613800138000"]
    },
    telegram: {
      accounts: {
        default: {
          botToken: "${TELEGRAM_BOT_TOKEN}"
        }
      }
    },
    whatsapp: {
      accounts: {
        default: {
          phoneNumber: "+8613800138000"
        }
      }
    }
  },

  cron: {
    enabled: true,
    sessionRetention: "24h"
  },

  hooks: {
    internal: {
      enabled: true,
      entries: {
        "session-memory": { enabled: true },
        "command-logger": { enabled: true }
      }
    }
  },

  browser: {
    enabled: false
  },

  sandbox: {
    enabled: false
  },

  logging: {
    level: "info",
    format: "text"
  }
}
```

## B.16 环境变量替换

配置支持环境变量替换：

```json5
{
  gateway: {
    auth: {
      token: "${GATEWAY_TOKEN}",      // 替换为 $GATEWAY_TOKEN
      password: "${GATEWAY_PASSWORD}"  // 替换为 $GATEWAY_PASSWORD
    }
  },
  channels: {
    telegram: {
      accounts: {
        default: {
          botToken: "${TELEGRAM_BOT_TOKEN}"
        }
      }
    }
  }
}
```

## B.17 SecretRef 配置

敏感信息可使用 SecretRef：

```json5
{
  models: {
    providers: {
      anthropic: {
        apiKey: {
          "$ref": "env:ANTHROPIC_API_KEY"
        }
      }
    }
  }
}
```

---

*返回：[附录 A：CLI 命令参考](appendix-a.md) | 下一章：[附录 C：API 参考](appendix-c.md)*
