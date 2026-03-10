# 附录 D：术语表

> 本附录提供 OpenClaw 相关术语的定义和解释。

## A

**A2UI (Automation to User Interface)**
自动化用户界面协议，用于在节点上渲染交互式 UI。

**ACP (Agent Communication Protocol)**
Agent 通信协议，IDE 与 Gateway 之间的桥接协议。

**Active Hours**
活跃时间配置，限制 Heartbeat 在指定时间窗口内运行。

**Agent**
代理，执行 AI 任务的核心组件，每个 Agent 有独立的工作区和会话。

**Agent Turn**
Agent 执行周期，包括接收输入、调用模型、执行工具、生成输出。

**Allowlist**
允许列表，用于 DM 策略、Exec 审批等场景的白名单机制。

**Announce Mode**
Cron 投递模式，直接将结果投递到指定频道。

**Auth Profile**
认证配置文件，存储模型提供商的 OAuth 或 API Key 凭证。

## B

**Background Process**
后台进程，Gateway 作为系统服务运行的模式。

**Backoff**
退避策略，失败后延迟重试的机制。

**Bindings**
路由绑定，将 Agent 绑定到特定通道或账号。

**Bootstrap Context**
引导上下文，Agent 启动时注入的文件和配置。

**Browser Profile**
浏览器配置文件，存储浏览器设置和用户数据。

## C

**Canvas**
画布，用于在节点上渲染 HTML/CSS/JS 内容的 WKWebView 容器。

**CDP (Chrome DevTools Protocol)**
Chrome 开发者工具协议，用于控制浏览器。

**Channel**
消息通道，如 WhatsApp、Telegram、Discord 等通信平台。

**ClawHub**
技能和插件的包管理和分发系统。

**Cron**
定时任务调度器，在指定时间执行 Agent 任务。

**Cron Job**
Cron 任务，包含调度配置和执行负载。

## D

**Dashboard**
Web 控制面板，提供 Gateway 状态和配置的可视化界面。

**Deep Link**
深度链接，`openclaw://` 协议用于触发 Agent 操作。

**Delivery Mode**
投递模式，控制 Cron 任务的输出投递方式。

**Device Auth**
设备认证，节点设备与 Gateway 配对的身份验证。

**Device Pairing**
设备配对，新设备连接 Gateway 时的审批流程。

**DM Policy**
私信策略，控制直接消息的处理方式（open/allowlist/deny）。

**Doctor**
健康检查工具，诊断和修复配置问题。

## E

**Elevated Tools**
提升权限的工具，需要额外审批才能执行。

**Environment Variables**
环境变量，用于配置敏感信息（如 API Key）。

**Exec Approvals**
执行审批，控制 `system.run` 等命令的执行权限。

**Event Stream**
事件流，WebSocket 实时推送的事件序列。

## F

**Fallback Model**
备用模型，主模型失败时的替代选项。

**Foreground Service**
前台服务，Android 应用保持连接持久的通知服务。

## G

**Gateway**
网关，OpenClaw 的核心服务，处理消息路由、Agent 调度等。

**Gateway Mode**
Gateway 模式，local（本地运行）或 remote（远程连接）。

**Group Policy**
群组策略，控制群组消息的处理方式。

## H

**Handler**
处理器，Hook 事件的执行逻辑。

**Heartbeat**
心跳监控，周期性执行 Agent Turn 进行背景监控。

**HEARTBEAT_OK**
心跳确认标记，表示无事需要关注，消息会被抑制。

**Hook**
钩子，响应系统事件的自动化脚本。

**Hook Pack**
钩子包，npm 包形式分发的多个 Hooks。

## I

**Isolated Session**
独立会话，Cron 任务在不污染主会话历史的独立环境中运行。

**Invoke**
调用，通过 CLI 调用节点命令。

## K

**Keyring**
钥匙串，macOS/iOS 存储凭证的安全容器。

## L

**Launchd**
macOS 服务管理器，用于管理 Gateway 后台进程。

**Legacy Config**
旧版配置，需要迁移的旧格式配置。

**Light Context**
轻量级上下文，仅加载 HEARTBEAT.md 的引导模式。

**Lobster**
工作流运行时，用于多步骤工具管道和审批流程。

**Loopback Binding**
本地回环绑定，最安全的网络绑定模式（仅 127.0.0.1）。

## M

**mDNS (Multicast DNS)**
多播 DNS，用于局域网设备发现（Bonjour）。

**MagicDNS**
Tailscale 的 DNS 服务，提供设备名称解析。

**Main Session**
主会话，Agent 的主要对话历史。

**Metadata**
元数据，SKILL.md/HOOK.md 中的配置信息。

**Model Alias**
模型别名，如 `opus` → `anthropic/claude-opus-4-6`。

## N

**Node**
节点，配套设备应用（如 iOS/Android/macOS 应用）。

**Node Host**
节点主机，运行在无头设备上的节点服务。

**NSD (Network Service Discovery)**
网络服务发现，Android 的 mDNS 实现。

## O

**Onboarding**
新手引导，首次使用时的配置向导。

**OpenClaw**
项目名称，开源 AI 助手网关系统。

## P

**Pairing Code**
配对码，设备配对时的验证码。

**Payload**
负载，Cron 任务执行的内容（systemEvent 或 agentTurn）。

**Plugin**
插件，扩展 OpenClaw 功能的模块。

**Preprocessed Message**
预处理消息，完成媒体/链接理解后的消息。

**Profile**
配置文件，命名配置集合（如多账号场景）。

## Q

**QR Login**
QR 码登录，WhatsApp 等通道的登录方式。

**Quiet Hours**
静默时间，Heartbeat 跳过的时间段。

## R

**Remote Mode**
远程模式，应用通过 SSH/Tailscale 连接远程 Gateway。

**Retry Policy**
重试策略，失败后的重试配置（次数、退避时间）。

**RPC (Remote Procedure Call)**
远程过程调用，WebSocket 请求 - 响应协议。

**Run History**
运行历史，Cron 任务的执行记录。

## S

**Sandbox**
沙箱，Docker 容器隔离工具执行的环境。

**Schedule**
调度配置，Cron 任务的时间规则（at/every/cron）。

**SecretRef**
密钥引用，引用环境变量中的敏感信息。

**SecretRef-aware**
支持 SecretRef 的命令，能读取和管理密钥引用。

**Session**
会话，Agent 对话的历史记录。

**Session Key**
会话键，会话的唯一标识符（如 `agent:main:main`）。

**Session Retention**
会话保留时间，Cron 独立会话的保留策略。

**Skill**
技能，预定义的功能模块（如 PDF 编辑、搜索等）。

**SRV Record**
SRV 记录，DNS 服务记录，用于跨网络发现。

**SSH Tunnel**
SSH 隧道，通过 SSH 转发 Gateway 端口。

**State Directory**
状态目录，存储会话、凭证、日志等数据（`~/.openclaw/`）。

**Stagger**
错开窗口，Cron 任务的时间偏移（避免并发压力）。

**Subagent**
子 Agent，主 Agent 调用的专用 Agent。

**Supervisor Config**
监督器配置，launchd/systemd/schtasks 服务配置。

**System Event**
系统事件，触发 Agent 处理的内部消息。

**Systemd**
Linux 服务管理器。

## T

**Tailscale**
零配置 VPN，用于跨网络设备连接。

**Tailscale Serve**
Tailscale 的本地服务暴露功能。

**Target**
投递目标，Heartbeat/Cron 消息的接收者。

**TCC Permission**
macOS 隐私权限（通知、辅助功能、屏幕录制等）。

**Terraform**
基础设施即代码工具，用于自动化 VPS 部署。

**Thinking Level**
思考级别，GPT-5.2 + Codex 模型的思考深度（off/minimal/low/medium/high/xhigh）。

**TLS Fingerprint**
TLS 指纹，证书固定用于安全连接。

**Tool**
工具，Agent 可调用的功能（如浏览器、文件操作等）。

**Tool Result Hook**
工具结果钩子，在工具结果持久化前进行修改。

**Transcript**
转录文件，会话对话的记录文件。

**Transcribed Message**
转录消息，音频转录后的消息。

## U

**Uninstall**
卸载，移除 OpenClaw 服务和数据。

## V

**Visibility Control**
可见性控制，Heartbeat 消息的投递规则（showOk/showAlerts/useIndicator）。

## W

**Wake Mode**
唤醒模式，Cron 任务的唤醒时机（now/next-heartbeat）。

**Webhook**
Web 钩子，外部系统通过 HTTP 触发 OpenClaw 任务。

**Workspace**
工作区，Agent 的文件和配置目录。

**Workspace Bootstrap**
工作区引导，Agent 启动时加载的文件（如 AGENTS.md、TOOLS.md）。

**WSL2**
Windows Subsystem for Linux 2，在 Windows 上运行 Linux。

## 其他

**`/` 命令**
Slash 命令，聊天中的特殊命令（如 `/new`、`/reset`、`/stop`）。

**`~/.openclaw/`**
OpenClaw 默认状态目录。

**`openclaw://`**
深度链接协议，用于触发本地操作。

**`HEARTBEAT.md`**
心跳清单文件，Heartbeat 运行时读取的任务列表。

**`BOOT.md`**
启动脚本文件，Gateway 启动时自动执行。

---

*返回：[附录 C：API 参考](appendix-c.md)*

---

## 书籍编写完成总结

《OpenClaw 完全指南》书籍已全部完成，包括：

### 主要内容
- **第 1-18 章**：完整的教程和技术文档
- **附录 A-D**：CLI 命令参考、配置 Schema、API 参考、术语表

### 章节列表
1. 认识 OpenClaw
2. 快速开始
3. 核心概念
4. 网关配置详解
5. 消息通道配置
6. 模型提供商配置
7. 安全与权限
8. 多代理路由
9. 工具系统
10. 技能开发
11. 插件开发
12. macOS 应用
13. iOS 节点
14. Android 节点
15. 远程访问
16. 部署与运维
17. 自动化
18. 故障排除

### 附录
- A. CLI 命令参考
- B. 配置 Schema
- C. API 参考
- D. 术语表

所有章节均遵循统一的模板格式，包含学习目标、前置条件、代码示例、表格对比和延伸阅读。
