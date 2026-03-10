# 第 13 章：iOS 节点

> 本章概述：讲解 OpenClaw iOS 应用作为节点的功能、配对、连接和 Canvas 能力。

## 学习目标

- 理解 iOS 节点的架构和功能
- 掌握配对和连接流程
- 学会使用 Canvas 和 A2UI
- 了解语音唤醒和对话模式
- 掌握常见故障排除

## 前置条件

- 已安装 iOS 应用（内部预览）
- Gateway 运行在另一设备上

---

## 13.1 iOS 节点概述

### 13.1.1 功能列表

iOS 应用作为 OpenClaw 的节点，暴露以下能力：

| 类别 | 功能 | 说明 |
|------|------|------|
| **Canvas** | WKWebView 渲染 | 网页画布显示 |
| **屏幕** | 屏幕快照 | 截图能力 |
| **相机** | 相机捕获 | 拍照/录像 |
| **位置** | GPS 定位 | 位置信息 |
| **语音** | 对话模式 | 语音交互 |
| **唤醒** | 语音唤醒 | 语音激活 |

### 13.1.2 运行要求

**Gateway 位置**：
- macOS 设备
- Linux 设备
- Windows（通过 WSL2）

**网络路径**（三选一）：
1. **同一 LAN**：通过 Bonjour mDNS
2. **Tailnet**：通过单播 DNS-SD
3. **手动主机/端口**：备用方案

---

## 13.2 快速开始

### 13.2.1 配对 + 连接流程

```
步骤 1：启动 Gateway
       ↓
步骤 2：iOS 应用选择 Gateway
       ↓
步骤 3：Gateway 主机批准配对
       ↓
步骤 4：验证连接
```

### 13.2.2 详细步骤

**步骤 1：启动 Gateway**

```bash
openclaw gateway --port 18789
```

**步骤 2：iOS 应用配置**

1. 打开 iOS 应用
2. 进入设置
3. 选择已发现的 Gateway，或启用手动主机

**步骤 3：批准配对请求**

```bash
# 查看待批准请求
openclaw nodes pending

# 批准
openclaw nodes approve <requestId>
```

**步骤 4：验证连接**

```bash
# 查看节点状态
openclaw nodes status

# 列出所有节点
openclaw gateway call node.list --params "{}"
```

---

## 13.3 设备发现

### 13.3.1 Bonjour（LAN 内）

Gateway 在局域网广播服务：

```
服务类型：_openclaw-gw._tcp
域名：local.
```

**iOS 行为**：
- 自动发现并列出 Bonjour Gateway
- 无需手动配置

### 13.3.2 Tailnet（跨网络）

当 mDNS 被阻止时，使用单播 DNS-SD：

**配置步骤**：

1. 选择域名（示例：`openclaw.internal.`）
2. 配置 Tailscale 分割 DNS
3. 设置 CoreDNS 示例（见 [Bonjour 文档](/gateway/bonjour)）

**DNS-SD 记录**：
```
_openclaw-gw._tcp.openclaw.internal.
```

### 13.3.3 手动主机/端口

**iOS 设置**：
1. 启用**手动主机**
2. 输入 Gateway 主机地址
3. 输入端口（默认 `18789`）

**适用场景**：
- Bonjour 不可用
- Tailnet 未配置
- 直接 IP 连接

---

## 13.4 Canvas + A2UI

### 13.4.1 Canvas 导航

iOS 节点渲染 WKWebView Canvas：

```bash
# 导航到 Canvas
openclaw nodes invoke \
  --node "iOS Node" \
  --command canvas.navigate \
  --params '{"url":"http://<gateway-host>:18789/__openclaw__/canvas/"}'
```

**Gateway Canvas 端点**：
| 路径 | 说明 |
|------|------|
| `/__openclaw__/canvas/` | Canvas 主页面 |
| `/__openclaw__/a2ui/` | A2UI 自动化界面 |

**自动导航**：
- 连接时自动导航到 A2UI（如果 Gateway 广播了 Canvas 主机 URL）
- 返回内置脚手架：`canvas.navigate` + `{"url":""}`

### 13.4.2 Canvas 执行

```bash
# 执行 JavaScript
openclaw nodes invoke \
  --node "iOS Node" \
  --command canvas.eval \
  --params '{"javaScript":"(() => { const {ctx} = window.__openclaw; ctx.clearRect(0,0,innerWidth,innerHeight); ctx.lineWidth=6; ctx.strokeStyle=\"#ff2d55\"; ctx.beginPath(); ctx.moveTo(40,40); ctx.lineTo(innerWidth-40, innerHeight-40); ctx.stroke(); return \"ok\"; })()"}'
```

### 13.4.3 Canvas 快照

```bash
# 获取 Canvas 快照
openclaw nodes invoke \
  --node "iOS Node" \
  --command canvas.snapshot \
  --params '{"maxWidth":900,"format":"jpeg"}'
```

**参数说明**：
| 参数 | 说明 | 默认值 |
|------|------|--------|
| `maxWidth` | 最大宽度（像素） | - |
| `format` | 图像格式 | `jpeg` |
| `quality` | JPEG 质量（0-1） | `0.8` |

---

## 13.5 语音功能

### 13.5.1 语音唤醒

**启用方式**：
- iOS 应用设置中启用
- 需要麦克风权限

**限制**：
- iOS 可能暂停后台音频
- 应用不活跃时功能受限

### 13.5.2 对话模式

**功能**：
- 持续语音交互
- 自动语音识别

**设置**：
- 在 iOS 应用中启用
- 需要稳定的网络连接

---

## 13.6 故障排除

### 13.6.1 常见错误

| 错误代码 | 原因 | 解决方案 |
|----------|------|----------|
| `NODE_BACKGROUND_UNAVAILABLE` | 应用在后台 | 将 iOS 应用带到前台 |
| `A2UI_HOST_NOT_CONFIGURED` | Gateway 未广播 Canvas 主机 | 检查 `canvasHost` 配置 |
| 配对提示未出现 | 自动批准失败 | 运行 `openclaw nodes pending` 手动批准 |
| 重新安装后重连失败 | 钥匙串令牌清除 | 重新配对节点 |

### 13.6.2 诊断命令

```bash
# 查看待批准设备
openclaw nodes pending

# 查看已连接节点
openclaw nodes status --connected

# 查看最近连接
openclaw nodes status --last-connected 24h

# 手动列出设备
openclaw devices list
```

### 13.6.3 连接问题排查

**排查流程**：

```
1. 检查 Gateway 是否运行
   ↓
2. 检查网络连通性（ping/telnet）
   ↓
3. 检查 Bonjour/Tailnet 发现
   ↓
4. 检查配对状态
   ↓
5. 查看 Gateway 日志
```

**网络测试**：
```bash
# 测试 Gateway 端口
telnet <gateway-host> 18789

# 测试 WebSocket
wscat -c ws://<gateway-host>:18789
```

---

## 13.7 配置示例

### 13.7.1 Gateway 配置

```json5
{
  gateway: {
    port: 18789,
    // Canvas 主机 URL（自动广播给 iOS 节点）
    canvasHost: "http://192.168.1.100:18789",

    // 配对设置
    pairing: {
      enabled: true,
      timeout: 3600  // 配对码有效期（秒）
    }
  }
}
```

### 13.7.2 iOS 应用设置

```
Gateway 连接:
□ 自动发现（Bonjour）
□ Tailnet DNS-SD
□ 手动主机：192.168.1.100:18789

语音功能:
□ 语音唤醒
□ 对话模式

权限:
□ 麦克风 ✓
□ 位置 ✓
□ 相机 ✓
```

---

## 13.8 高级用法

### 13.8.1 远程相机快照

```bash
# 触发 iOS 相机拍照
openclaw nodes invoke \
  --node "iOS Node" \
  --command camera.snap \
  --params '{}'
```

### 13.8.2 屏幕录制

```bash
# 开始录屏
openclaw nodes invoke \
  --node "iOS Node" \
  --command screen.record \
  --params '{"duration": 30}'
```

### 13.8.3 位置获取

```bash
# 获取当前位置
openclaw nodes invoke \
  --node "iOS Node" \
  --command location.get \
  --params '{}'
```

---

## 13.9 开发注意事项

### 13.9.1 内部预览状态

- iOS 应用处于**内部预览**阶段
- 尚未公开分发
- 功能可能变更

### 13.9.2 从源码运行

```bash
# 克隆仓库
git clone <repo>

# 进入 iOS 目录
cd apps/ios

# 打开 Xcode
open OpenClaw.xcodeproj

# 或使用 Xcode Cloud 构建
```

### 13.9.3 钥匙串配对令牌

- 配对令牌存储在 iOS 钥匙串
- 重新安装应用会清除令牌
- 需要重新配对

---

## 本章小结

- **iOS 节点**：连接 Gateway，暴露 Canvas/相机/屏幕/位置能力
- **发现方式**：Bonjour（LAN）、Tailnet（跨网络）、手动主机
- **配对流程**：iOS 发起 → Gateway 批准 → 建立连接
- **Canvas 能力**：WKWebView 渲染、JS 执行、快照
- **语音功能**：语音唤醒、对话模式（后台受限）
- **常见错误**：前台要求、Canvas 配置、配对令牌

## 延伸阅读

- [配对机制](https://docs.openclaw.ai/gateway/pairing)
- [设备发现](https://docs.openclaw.ai/gateway/discovery)
- [Bonjour 配置](https://docs.openclaw.ai/gateway/bonjour)
- [第 14 章：Android 节点](chapter-14.md)

---

*上一章：[第 12 章：macOS 应用](chapter-12.md) | 下一章：[第 14 章：Android 节点](chapter-14.md)*
