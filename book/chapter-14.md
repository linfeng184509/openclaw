# 第 14 章：Android 节点

> 本章概述：讲解 OpenClaw Android 应用作为节点的功能、连接、聊天、Canvas 和语音能力。

## 学习目标

- 理解 Android 节点的架构和角色
- 掌握连接和配对流程
- 学会使用聊天、Canvas 和相机命令
- 了解语音功能和设备命令
- 掌握跨网络发现和故障排除

## 前置条件

- 已安装 Android 应用
- Gateway 运行在另一设备上

---

## 14.1 Android 节点概述

### 14.1.1 角色定位

| 属性 | 说明 |
|------|------|
| **角色** | 配套节点应用（不托管 Gateway） |
| **需要 Gateway** | 是（macOS/Linux/Windows WSL2） |
| **连接协议** | WebSocket + mDNS/NSD |
| **前台服务** | 持久通知保持连接 |

### 14.1.2 支持的功能

| 类别 | 功能 |
|------|------|
| **聊天** | 会话选择、历史记录、推送更新 |
| **Canvas** | WKWebView 渲染、JS 执行、快照 |
| **相机** | 拍照（jpg）、录像（mp4） |
| **语音** | 语音转录、TTS 播放 |
| **设备** | 状态、权限、健康检查 |
| **通知** | 列表、操作 |
| **照片** | 获取最新照片 |
| **联系人** | 搜索、添加 |
| **日历** | 事件、添加事件 |
| **运动** | 活动检测、计步器 |

---

## 14.2 连接流程

### 14.2.1 架构概览

```
Android 节点应用 ⇄ (mDNS/NSD + WebSocket) ⇄ Gateway 网关
```

- Android 直接连接 Gateway WebSocket（默认 `ws://<host>:18789`）
- 使用设备配对（`role: node`）
- 通过前台服务保持连接活跃

### 14.2.2 前置条件

**Gateway 主机**：
- 可以在"主"设备上运行 Gateway
- 可以运行 CLI（`openclaw`）

**网络要求**（三选一）：
1. **同一 LAN**：mDNS/NSD 发现
2. **Tailscale tailnet**：单播 DNS-SD
3. **手动主机/端口**：备用方案

### 14.2.3 连接步骤

```
步骤 1：启动 Gateway
       ↓
步骤 2：验证发现（可选）
       ↓
步骤 3：Android 连接
       ↓
步骤 4：批准配对
       ↓
步骤 5：验证连接
```

---

## 14.3 详细配置

### 14.3.1 启动 Gateway

```bash
openclaw gateway --port 18789 --verbose
```

**确认日志**：
```
listening on ws://0.0.0.0:18789
```

**仅 tailnet 配置**（跨网络推荐）：

```json5
{
  gateway: {
    bind: "tailnet"  // 绑定到 tailnet IP
  }
}
```

### 14.3.2 验证发现（可选）

**LAN 发现**：
```bash
dns-sd -B _openclaw-gw._tcp local.
```

**Tailnet 发现**（跨网络）：

1. 设置 DNS-SD 区域（示例：`openclaw.internal.`）
2. 发布 `_openclaw-gw._tcp` 记录
3. 配置 Tailscale 分割 DNS

详见 [Bonjour 文档](/gateway/bonjour)。

### 14.3.3 Android 连接

**应用内操作**：

1. 打开**连接**标签页
2. 使用**设置码**或**手动**模式
3. 如果发现被阻止，在**高级控制**中输入主机/端口

**自动重连**：
- 首次配对后，Android 启动时自动重连
- 优先使用手动端点（如果启用）
- 否则尝试最后发现的 Gateway

### 14.3.4 批准配对

```bash
# 查看待批准设备
openclaw devices list

# 批准
openclaw devices approve <requestId>

# 拒绝
openclaw devices reject <requestId>
```

### 14.3.5 验证连接

```bash
# 节点状态
openclaw nodes status

# Gateway 节点列表
openclaw gateway call node.list --params "{}"
```

---

## 14.4 聊天功能

### 14.4.1 会话选择

Android 聊天标签页支持会话选择：
- 默认：`main` 会话
- 可选：其他现有会话

### 14.4.2 聊天命令

| 命令 | 说明 | 返回 |
|------|------|------|
| `chat.history` | 获取历史消息 | 消息列表 |
| `chat.send` | 发送消息 | 发送状态 |
| `chat.subscribe` | 订阅推送更新 | 事件流 |

**推送更新**：
- 事件类型：`chat`
- 尽力而为交付

### 14.4.3 跨客户端一致性

Android 使用 Gateway 的**主会话键**（`main`）：
- 历史与 WebChat 共享
- 与其他客户端同步

---

## 14.5 Canvas + 相机

### 14.5.1 Gateway Canvas 主机

**用途**：显示 Agent 可编辑的 HTML/CSS/JS 内容

**配置步骤**：

1. 在 Gateway 主机创建文件：
   ```
   ~/.openclaw/workspace/canvas/index.html
   ```

2. 导航 Android 节点（LAN）：
   ```bash
   openclaw nodes invoke \
     --node "<Android Node>" \
     --command canvas.navigate \
     --params '{"url":"http://<gateway-hostname>.local:18789/__openclaw__/canvas/"}'
   ```

**Tailnet 选项**：
- 使用 MagicDNS 名称或 tailnet IP
- 示例：`http://<gateway-magicdns>:18789/__openclaw__/canvas/`

**特性**：
- 实时重载客户端注入 HTML
- 文件更改时自动重载
- A2UI 主机：`/__openclaw__/a2ui/`

### 14.5.2 Canvas 命令

| 命令 | 说明 | 参数 |
|------|------|------|
| `canvas.eval` | 执行 JS | `{"javaScript": "..."}` |
| `canvas.snapshot` | 获取快照 | `{"format":"jpeg","maxWidth":900}` |
| `canvas.navigate` | 导航 URL | `{"url":"..."}` |
| `canvas.a2ui.push` | A2UI 推送 | JSONL 数据 |
| `canvas.a2ui.reset` | A2UI 重置 | - |

**返回格式**：
```json
{
  "format": "jpeg",
  "base64": "<base64 图像数据>"
}
```

**返回默认脚手架**：
```bash
canvas.navigate --params '{"url":""}'
# 或
canvas.navigate --params '{"url":"/"}'
```

### 14.5.3 相机命令

| 命令 | 格式 | 要求 |
|------|------|------|
| `camera.snap` | jpg | 前台 + 相机权限 |
| `camera.clip` | mp4 | 前台 + 相机权限 |

**限制**：
- 仅前台可用
- 需要相机权限

---

## 14.6 语音功能

### 14.6.1 语音标签页

Android 语音功能特点：
- **单麦克风开关**流程
- **语音转录**捕获
- **TTS 播放**（ElevenLabs 或系统 TTS）

### 14.6.2 限制

- 语音在应用离开前台时停止
- 语音唤醒/对话模式切换目前在 Android UX 中移除

---

## 14.7 设备命令

### 14.7.1 设备状态

```bash
# 设备状态
openclaw nodes invoke \
  --node "<Android Node>" \
  --command device.status \
  --params '{}'

# 设备信息
openclaw nodes invoke \
  --node "<Android Node>" \
  --command device.info \
  --params '{}'

# 权限状态
openclaw nodes invoke \
  --node "<Android Node>" \
  --command device.permissions \
  --params '{}'

# 健康检查
openclaw nodes invoke \
  --node "<Android Node>" \
  --command device.health \
  --params '{}'
```

### 14.7.2 通知操作

```bash
# 列出通知
openclaw nodes invoke \
  --node "<Android Node>" \
  --command notifications.list \
  --params '{}'

# 执行通知操作
openclaw nodes invoke \
  --node "<Android Node>" \
  --command notifications.actions \
  --params '{"notificationId": "123", "actionIndex": 0}'
```

### 14.7.3 照片

```bash
# 获取最新照片
openclaw nodes invoke \
  --node "<Android Node>" \
  --command photos.latest \
  --params '{}'
```

### 14.7.4 联系人

```bash
# 搜索联系人
openclaw nodes invoke \
  --node "<Android Node>" \
  --command contacts.search \
  --params '{"query": "John"}'

# 添加联系人
openclaw nodes invoke \
  --node "<Android Node>" \
  --command contacts.add \
  --params '{"name": "John", "phone": "+1234567890"}'
```

### 14.7.5 日历

```bash
# 获取事件
openclaw nodes invoke \
  --node "<Android Node>" \
  --command calendar.events \
  --params '{"start": "2026-03-10", "end": "2026-03-17"}'

# 添加事件
openclaw nodes invoke \
  --node "<Android Node>" \
  --command calendar.add \
  --params '{"title": "会议", "start": "2026-03-11T10:00:00Z"}'
```

### 14.7.6 运动传感器

```bash
# 活动检测
openclaw nodes invoke \
  --node "<Android Node>" \
  --command motion.activity \
  --params '{}'

# 计步器
openclaw nodes invoke \
  --node "<Android Node>" \
  --command motion.pedometer \
  --params '{}'
```

---

## 14.8 跨网络配置

### 14.8.1 Tailscale 设置

**维也纳 ⇄ 伦敦场景**：

1. **Gateway 主机配置**：
   ```json5
   {
     gateway: {
       bind: "tailnet"
     }
   }
   ```

2. **DNS-SD 区域设置**：
   - 域名：`openclaw.internal.`
   - 记录：`_openclaw-gw._tcp`

3. **Tailscale 分割 DNS**：
   - 将域名指向 DNS 服务器

### 14.8.2 MagicDNS 使用

```bash
# 使用 MagicDNS 名称
openclaw nodes invoke \
  --node "<Android Node>" \
  --command canvas.navigate \
  --params '{"url":"http://gateway-hostname.ts.net:18789/__openclaw__/canvas/"}'

# 或使用 tailnet IP
openclaw nodes invoke \
  --node "<Android Node>" \
  --command canvas.navigate \
  --params '{"url":"http://100.x.y.z:18789/__openclaw__/canvas/"}'
```

---

## 14.9 故障排除

### 14.9.1 常见问题

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 无法发现 Gateway | mDNS 被阻止 | 使用手动主机或 DNS-SD |
| 配对的请求未出现 | 自动批准失败 | CLI 手动批准 |
| 连接断开 | 网络不稳定 | 检查前台服务状态 |
| Canvas 不加载 | URL 错误 | 检查 `canvasHost` 配置 |
| 相机命令失败 | 权限不足 | 授予相机权限 |

### 14.9.2 诊断命令

```bash
# 查看节点列表
openclaw nodes list

# 查看连接状态
openclaw nodes status --connected

# 查看最近连接
openclaw nodes status --last-connected 24h

# 测试 WebSocket
wscat -c ws://<gateway-host>:18789
```

### 14.9.3 日志分析

**Gateway 日志**：
```bash
openclaw gateway --verbose
```

**查找内容**：
- `listening on ws://...`
- `node connected: <id>`
- `pairing request: <requestId>`

---

## 本章小结

- **Android 角色**：配套节点，不托管 Gateway
- **连接方式**：WebSocket + mDNS/NSD/Tailnet
- **前台服务**：持久通知保持连接
- **聊天功能**：会话选择、历史同步、推送更新
- **Canvas 能力**：HTML 渲染、JS 执行、实时重载
- **相机命令**：拍照/录像（前台 + 权限）
- **语音功能**：转录 + TTS（前台限制）
- **设备命令**：状态、通知、照片、联系人、日历、运动

## 延伸阅读

- [Gateway 配对](https://docs.openclaw.ai/gateway/pairing)
- [设备发现](https://docs.openclaw.ai/gateway/discovery)
- [相机节点](https://docs.openclaw.ai/nodes/camera)
- [第 15 章：远程访问](chapter-15.md)

---

*上一章：[第 13 章：iOS 节点](chapter-13.md) | 下一章：[第 15 章：远程访问](chapter-15.md)*
