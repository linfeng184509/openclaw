# 第 16 章：部署与运维

> 本章概述：讲解 OpenClaw 的生产环境部署方案，包括 Docker 容器化、VPS 部署、系统配置、监控和故障排除。

## 学习目标

- 掌握 Docker 容器化部署
- 学会 VPS 生产环境配置
- 了解系统管理和监控
- 掌握故障排除方法
- 了解安全最佳实践

## 前置条件

- 了解 Docker 基础
- 有 Linux 服务器使用经验
- 了解基础网络配置

---

## 16.1 部署方案概述

### 16.1.1 部署选项对比

| 方案 | 适用场景 | 复杂度 | 成本 |
|------|----------|--------|------|
| **本地安装** | 个人开发/测试 | 低 | 低 |
| **Docker 容器** | 隔离环境/测试 | 中 | 低 |
| **VPS 部署** | 生产/24/7 运行 | 中 | 中 |
| **Kubernetes** | 大规模部署 | 高 | 高 |

### 16.1.2 推荐架构

**个人使用**：
```
笔记本电脑 → 本地 Gateway →  channels/macOS 应用
```

**生产使用**：
```
VPS（24/7 Gateway）← SSH/Tailscale → 多客户端
                          ↓
                    iOS/Android 节点
```

---

## 16.2 Docker 容器化部署

### 16.2.1 前置要求

| 要求 | 说明 |
|------|------|
| Docker Desktop/Engine | Docker Compose v2+ |
| 内存 | 至少 2GB（构建时可能需要更多） |
| 磁盘 | 足够存放镜像和日志 |
| 网络 | 如需外网访问，配置防火墙 |

### 16.2.2 快速启动

**使用脚本（推荐）**：
```bash
# 克隆仓库
git clone https://github.com/openclaw/openclaw.git
cd openclaw

# 运行设置脚本
./docker-setup.sh
```

**脚本自动完成**：
- 构建 Gateway 镜像
- 运行 onboarding 向导
- 生成 Gateway token
- 启动 Gateway 服务

**访问 Control UI**：
```
http://127.0.0.1:18789/
```

### 16.2.3 环境变量配置

**可选环境变量**：
```bash
# 使用远程镜像（跳过本地构建）
export OPENCLAW_IMAGE="ghcr.io/openclaw/openclaw:latest"

# 安装额外 apt 包
export OPENCLAW_DOCKER_APT_PACKAGES="ffmpeg build-essential"

# 预安装扩展依赖
export OPENCLAW_EXTENSIONS="diagnostics-otel matrix"

# 额外挂载目录
export OPENCLAW_EXTRA_MOUNTS="$HOME/.codex:/home/node/.codex:ro"

# 持久化 /home/node
export OPENCLAW_HOME_VOLUME="openclaw_home"

# 启用沙箱
export OPENCLAW_SANDBOX=1
```

### 16.2.4 Docker Compose 配置

**基础配置**：
```yaml
services:
  openclaw-gateway:
    image: openclaw:local
    build: .
    restart: unless-stopped
    env_file:
      - .env
    environment:
      - HOME=/home/node
      - NODE_ENV=production
      - OPENCLAW_GATEWAY_BIND=lan
      - OPENCLAW_GATEWAY_PORT=18789
      - OPENCLAW_GATEWAY_TOKEN=change-me-now
    volumes:
      - ${OPENCLAW_CONFIG_DIR}:/home/node/.openclaw
      - ${OPENCLAW_WORKSPACE_DIR}:/home/node/.openclaw/workspace
    ports:
      - "127.0.0.1:18789:18789"
    command:
      - node
      - dist/index.js
      - gateway
      - --bind
      - lan
      - --port
      - 18789
```

### 16.2.5 持久化数据

**必须持久化的目录**：
| 组件 | 位置 | 说明 |
|------|------|------|
| Gateway 配置 | `/home/node/.openclaw/` | 含 tokens、OAuth |
| Skill 配置 | `/home/node/.openclaw/skills/` | Skill 状态 |
| Agent 工作区 | `/home/node/.openclaw/workspace/` | 代码和工件 |
| WhatsApp 会话 | `/home/node/.openclaw/` | QR 登录持久化 |
| Gmail keyring | `/home/node/.openclaw/` | 需要密码 |

**外部二进制**（必须在镜像中构建）：
- `gog` - Gmail CLI
- `goplaces` - Google Places CLI
- `wacli` - WhatsApp CLI

### 16.2.6 Dockerfile 示例

```dockerfile
FROM node:22-bookworm

# 安装系统依赖
RUN apt-get update && apt-get install -y socat && rm -rf /var/lib/apt/lists/*

# 安装 Gmail CLI
RUN curl -L https://github.com/steipete/gog/releases/latest/download/gog_Linux_x86_64.tar.gz \
  | tar -xz -C /usr/local/bin && chmod +x /usr/local/bin/gog

# 安装 Google Places CLI
RUN curl -L https://github.com/steipete/goplaces/releases/latest/download/goplaces_Linux_x86_64.tar.gz \
  | tar -xz -C /usr/local/bin && chmod +x /usr/local/bin/goplaces

# 安装 WhatsApp CLI
RUN curl -L https://github.com/steipete/wacli/releases/latest/download/wacli_Linux_x86_64.tar.gz \
  | tar -xz -C /usr/local/bin && chmod +x /usr/local/bin/wacli

WORKDIR /app
COPY package.json pnpm-lock.yaml pnpm-workspace.yaml .npmrc ./
COPY ui/package.json ./ui/package.json
COPY scripts ./scripts

RUN corepack enable
RUN pnpm install --frozen-lockfile

COPY . .
RUN pnpm build
RUN pnpm ui:install
RUN pnpm ui:build

ENV NODE_ENV=production
CMD ["node","dist/index.js"]
```

### 16.2.7 构建和启动

```bash
# 构建镜像
docker compose build

# 启动服务
docker compose up -d openclaw-gateway

# 验证二进制
docker compose exec openclaw-gateway which gog
docker compose exec openclaw-gateway which goplaces
docker compose exec openclaw-gateway which wacli

# 查看日志
docker compose logs -f openclaw-gateway
```

### 16.2.8 健康检查

```bash
# 简单探针
curl -fsS http://127.0.0.1:18789/healthz
curl -fsS http://127.0.0.1:18789/readyz

# 认证健康检查
docker compose exec openclaw-gateway \
  node dist/index.js health --token "$OPENCLAW_GATEWAY_TOKEN"
```

---

## 16.3 VPS 生产部署

### 16.3.1 选择 VPS 提供商

**推荐提供商**：
| 提供商 | 起始价格 | 说明 |
|--------|----------|------|
| Hetzner | ~$5/月 | 欧洲为主，性价比高 |
| Exe.dev | ~$5/月 | 简单易用 |
| DigitalOcean | ~$6/月 | 全球覆盖 |
| Linode | ~$5/月 | 全球覆盖 |

### 16.3.2 服务器初始化

```bash
# SSH 登录
ssh root@YOUR_VPS_IP

# 更新系统
apt-get update
apt-get upgrade -y

# 安装基础工具
apt-get install -y git curl ca-certificates

# 安装 Docker
curl -fsSL https://get.docker.com | sh

# 验证安装
docker --version
docker compose version
```

### 16.3.3 创建持久目录

```bash
# 创建目录
mkdir -p /root/.openclaw/workspace

# 设置权限（容器用户 uid 1000）
chown -R 1000:1000 /root/.openclaw
```

### 16.3.4 配置环境变量

创建 `.env` 文件：
```bash
OPENCLAW_IMAGE=openclaw:latest
OPENCLAW_GATEWAY_TOKEN=<强随机 token>
OPENCLAW_GATEWAY_BIND=lan
OPENCLAW_GATEWAY_PORT=18789

OPENCLAW_CONFIG_DIR=/root/.openclaw
OPENCLAW_WORKSPACE_DIR=/root/.openclaw/workspace

GOG_KEYRING_PASSWORD=<强随机密码>
XDG_CONFIG_HOME=/home/node/.openclaw
```

**生成强随机值**：
```bash
openssl rand -hex 32
```

### 16.3.5 Terraform 自动化（可选）

**社区维护的 Terraform 配置**：
- [openclaw-terraform-hetzner](https://github.com/andreesg/openclaw-terraform-hetzner)
- [openclaw-docker-config](https://github.com/andreesg/openclaw-docker-config)

**功能**：
- 模块化 Terraform 配置
- 云初始化自动配置
- 部署脚本（bootstrap/deploy/backup）
- 安全加固（防火墙、UFW、SSH _only）

### 16.3.6 SSH 隧道访问

**从本地创建隧道**：
```bash
ssh -N -L 18789:127.0.0.1:18789 root@YOUR_VPS_IP
```

**访问 Web UI**：
```
http://127.0.0.1:18789/
```

### 16.3.7 Tailscale 集成（推荐）

**在 VPS 上安装 Tailscale**：
```bash
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up
```

**启用 Tailscale Serve**：
```bash
tailscale serve 18789
```

**获取访问 URL**：
```bash
tailscale status
# 输出：https://gateway-hostname.ts.net:443
```

---

## 16.4 沙箱配置

### 16.4.1 沙箱模式

| 模式 | 说明 |
|------|------|
| `off` | 禁用沙箱 |
| `non-main` | 仅非主会话使用沙箱（推荐） |
| `all` | 所有会话使用沙箱 |

### 16.4.2 沙箱范围

| 范围 | 说明 |
|------|------|
| `session` | 每会话一个容器 |
| `agent` | 每代理一个容器（默认） |
| `shared` | 所有会话共享容器 |

### 16.4.3 沙箱配置示例

```json5
{
  agents: {
    defaults: {
      sandbox: {
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
  },
  tools: {
    sandbox: {
      tools: {
        allow: ["exec", "process", "read", "write", "edit"],
        deny: ["browser", "canvas", "nodes", "cron"]
      }
    }
  }
}
```

### 16.4.4 构建沙箱镜像

```bash
# 基础沙箱镜像
scripts/sandbox-setup.sh

# 带常用工具的镜像
scripts/sandbox-common-setup.sh

# 沙箱浏览器镜像
scripts/sandbox-browser-setup.sh
```

---

## 16.5 系统管理

### 16.5.1 服务控制

```bash
# 查看状态
openclaw gateway status
openclaw status

# 启动服务
openclaw gateway start

# 停止服务
openclaw gateway stop

# 重启服务
openclaw gateway restart

# 安装服务
openclaw gateway install

# 卸载服务
openclaw gateway uninstall
```

### 16.5.2 日志管理

```bash
# 查看日志
openclaw logs --follow

# 查看最近日志
openclaw logs --limit 100

# 查看错误日志
openclaw logs --level error
```

### 16.5.3 配置管理

```bash
# 查看配置
openclaw config get gateway

# 设置配置
openclaw config set gateway.auth.token "new-token"

# 导出配置
openclaw config export > backup.json

# 导入配置
openclaw config import backup.json
```

### 16.5.4 设备管理

```bash
# 列出设备
openclaw devices list

# 批准设备
openclaw devices approve <requestId>

# 拒绝设备
openclaw devices reject <requestId>

# 查看已批准设备
openclaw devices approved
```

---

## 16.6 监控和告警

### 16.6.1 健康检查端点

| 端点 | 说明 | 认证 |
|------|------|------|
| `/healthz` | 存活探针 | 无需 |
| `/readyz` | 就绪探针 | 无需 |
| `/health` | 完整健康 | 需要 |

### 16.6.2 监控指标

**关键指标**：
- Gateway 运行状态
- 通道连接状态
- 活跃会话数
- 内存使用
- CPU 使用
- 请求延迟

### 16.6.3 日志收集

**日志位置**：
- Docker：`docker compose logs`
- 系统：`~/.openclaw/logs/gateway.log`
- 错误：`~/.openclaw/logs/gateway.err.log`

---

## 16.7 故障排除

### 16.7.1 诊断命令阶梯

```bash
# 1. 基础状态
openclaw status

# 2. Gateway 状态
openclaw gateway status

# 3. 查看日志
openclaw logs --follow

# 4. 医生诊断
openclaw doctor

# 5. 通道探测
openclaw channels status --probe
```

### 16.7.2 常见问题

| 问题 | 可能原因 | 解决方案 |
|------|----------|----------|
| `EADDRINUSE` | 端口冲突 | 更改端口或停止占用进程 |
| `refusing to bind without auth` | 非 loopback 绑定未配置认证 | 配置 token/password |
| `Gateway start blocked` | `gateway.mode` 不是 `local` | 设置 `gateway.mode=local` |
| `unauthorized` | token/password 不匹配 | 检查认证配置 |
| `pairing required` | 设备未批准 | 运行 `openclaw devices approve` |
| `exit 127` | CLI 未找到 | 确保 `openclaw` 在 PATH 中 |

### 16.7.3 通道问题

**无回复**：
```bash
# 检查配对
openclaw pairing list --channel <channel>

# 检查配置
openclaw config get channels

# 查看日志
openclaw logs --follow
```

**常见日志信号**：
- `drop guild message (mention required)` → 需要提及
- `pairing request` → 需要批准配对
- `blocked/allowlist` → 发送者被过滤

### 16.7.4 Docker 问题

**镜像构建失败（OOM）**：
```bash
# 增加内存或添加 swap
# 或使用远程镜像
export OPENCLAW_IMAGE="ghcr.io/openclaw/openclaw:latest"
```

**权限错误**：
```bash
# Linux 主机修复权限
sudo chown -R 1000:1000 /path/to/openclaw-config
```

**二进制丢失**：
```bash
# 重新构建镜像
docker compose build
docker compose up -d
```

---

## 16.8 安全最佳实践

### 16.8.1 网络暴露

**推荐配置**：
```json5
{
  gateway: {
    bind: "loopback"  // 仅 loopback + SSH/Tailscale
  }
}
```

**非 loopback 绑定必须配置认证**：
```json5
{
  gateway: {
    bind: "lan",
    auth: {
      token: "<强随机 token>"
    }
  }
}
```

### 16.8.2 凭证管理

**环境变量优先级**：
```
显式（--token/--password） > 配置 > 环境变量
```

**推荐做法**：
- 使用环境变量存储敏感信息
- 不要提交 `.env` 文件到版本控制
- 定期轮换 token

### 16.8.3 Docker 安全

**安全配置**：
```yaml
services:
  openclaw-gateway:
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - NET_RAW
      - NET_ADMIN
    read_only: true  # 如需可写，挂载特定目录
```

### 16.8.4 防火墙配置

**UFW 配置示例**：
```bash
# 允许 SSH
ufw allow 22/tcp

# 允许 Tailscale
ufw allow from 100.64.0.0/10

# 启用防火墙
ufw enable
```

**Docker 防火墙**：
```bash
# 防止 Docker 绕过防火墙
iptables -I DOCKER-USER -i eth0 -j DROP
iptables -I DOCKER-USER -i eth0 -p tcp --dport 22 -j ACCEPT
iptables -I DOCKER-USER -i eth0 -p tcp --dport 443 -j ACCEPT
```

---

## 16.9 备份和恢复

### 16.9.1 备份数据

**备份目录**：
```bash
# 创建备份
tar -czf openclaw-backup-$(date +%Y%m%d).tar.gz \
  ~/.openclaw/
```

**备份内容**：
- `~/.openclaw/openclaw.json` - 配置
- `~/.openclaw/agents/` - Agent 状态和会话
- `~/.openclaw/workspace/` - 工作区文件
- `~/.openclaw/skills/` - 技能配置

### 16.9.2 恢复数据

```bash
# 停止服务
openclaw gateway stop

# 恢复数据
tar -xzf openclaw-backup-YYYYMMDD.tar.gz -C ~/

# 启动服务
openclaw gateway start
```

### 16.9.3 迁移到另一台服务器

```bash
# 在旧服务器备份
tar -czf openclaw-backup.tar.gz ~/.openclaw/

# 传输到新服务器
scp openclaw-backup.tar.gz user@new-server:~/

# 在新服务器恢复
tar -xzf openclaw-backup.tar.gz -C ~/
```

---

## 本章小结

- **Docker 部署**：容器化、隔离、易迁移
- **VPS 部署**：24/7 运行、多客户端访问
- **沙箱隔离**：Docker 容器运行工具，减少风险
- **监控**：健康检查端点、日志、指标
- **故障排除**：诊断命令阶梯、常见问题模式
- **安全**：loopback 绑定优先、认证必须、防火墙配置
- **备份**：定期备份 `~/.openclaw/`，可迁移恢复

## 延伸阅读

- [Docker 详解](https://docs.openclaw.ai/install/docker)
- [Hetzner VPS 部署](https://docs.openclaw.ai/install/hetzner)
- [沙箱详解](https://docs.openclaw.ai/gateway/sandboxing)
- [安全指南](https://docs.openclaw.ai/gateway/security)
- [第 17 章：自动化](chapter-17.md)

---

*上一章：[第 15 章：远程访问](chapter-15.md) | 下一章：[第 17 章：自动化](chapter-17.md)*
