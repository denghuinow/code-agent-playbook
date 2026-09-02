# ChatGPT DevSpace 与 Agent Orchestrator 接入指南

本文档介绍如何将 DevSpace 和 Agent Orchestrator 通过 Cloudflare Tunnel 暴露给 ChatGPT，实现两种不同的 AI Agent 协作模式。

## 目录

- [架构概览](#架构概览)
- [组件职责分工](#组件职责分工)
- [DevSpace 部署](#devspace-部署)
- [Agent Orchestrator + Supergateway 部署](#agent-orchestrator--supergateway-部署)
- [Cloudflare Tunnel 配置](#cloudflare-tunnel-配置)
- [ChatGPT MCP Endpoint 配置](#chatgpt-mcp-endpoint-配置)
- [测试与验收](#测试与验收)
- [工作流选择建议](#工作流选择建议)
- [常见故障排查](#常见故障排查)
- [安全提示](#安全提示)

---

## 架构概览

### 数据流架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              ChatGPT                                        │
│  ┌─────────────────┐  ┌─────────────────┐                                  │
│  │  DevSpace MCP   │  │ Agent Orchestr. │                                  │
│  │   (HTTP)        │  │  MCP (Streamable│                                  │
│  └────────┬────────┘  └────────┬────────┘                                  │
└───────────┼────────────────────┼───────────────────────────────────────────┘
            │                    │
            ▼                    ▼
┌───────────────────┐  ┌─────────────────────────────────┐
│ Cloudflare Tunnel │  │      Cloudflare Tunnel          │
│  (HTTPS/WSS)      │  │      (HTTPS/WSS)                │
│  devspace.example │  │      agent.example.com           │
└─────────┬─────────┘  └──────────────┬──────────────────┘
          │                          │
          │  HTTP/REST              │  Streamable HTTP
          ▼                          ▼
┌───────────────────┐  ┌─────────────────────────────────┐
│   DevSpace        │  │         Supergateway             │
│   (HTTP MCP)      │  │  (stdio ↔ Streamable HTTP)      │
│   Port: 3100      │  │  Port: 3080                      │
└───────────────────┘  └──────────────┬──────────────────┘
                                     │
                                     │  stdio
                                     ▼
                           ┌─────────────────────┐
                           │  Agent Orchestrator │
                           │  (Claude Worker)    │
                           └─────────────────────┘
```

### 两种模式对比

| 维度 | DevSpace | Agent Orchestrator |
|------|----------|-------------------|
| MCP 协议 | HTTP (原生) | stdio (转 Streamable HTTP) |
| 连接方式 | Direct | 经 Supergateway 中转 |
| Worker 类型 | 单次请求 | 持久 Worker |
| 典型用途 | 读代码、git diff、简短构建 | 长任务、SSH、iperf、持续监控 |
| 会话持久性 | 请求级 | Worker 级 |

---

## 组件职责分工

### DevSpace

- **协议**：原生 HTTP MCP，无须中转
- **职责**：接收 ChatGPT 请求，执行源码读取、文件修改、git 操作、短时构建测试
- **端口**：3100
- **适用场景**：需要快速交互的轻量任务

### Agent Orchestrator

- **协议**：stdio MCP，通过 Supergateway 转为 Streamable HTTP
- **职责**：管理持久 Claude Worker，执行长时间运行的构建、部署、监控任务
- **端口**：3080 (Supergateway)
- **适用场景**：构建耗时任务、SSH 远程操作、网络性能测试、持续集成

### Supergateway

- **职责**：将 Agent Orchestrator 的 stdio MCP 协议转换为 Streamable HTTP，使其可通过 Cloudflare Tunnel 暴露
- **端口**：3080
- **注意**：需要配置 `IS_SANDBOX=1` 环境变量以支持 root 环境下运行 Claude Worker

### Cloudflare Tunnel

- **职责**：将内网服务安全暴露到公网，提供 HTTPS/WSS 加密通道
- **两种配置方式**：
  - Ingress Rule 配置（推荐用于固定服务）
  - `cloudflared tunnel run` 直接运行

---

## DevSpace 部署

### 1. 安装 DevSpace

```bash
# 方式一：Docker 运行
docker run -d \
  --name devspace \
  -p 3100:3100 \
  -v /home/user/project/my-project:/workspace \
  -e PROJECT_PATH=/workspace \
  devspace:latest

# 方式二：直接运行（假设已编译）
./devspace --port 3100 --project /home/user/project/my-project
```

### 2. 配置 Cloudflare Tunnel Ingress

```yaml
# tunnel.yaml
tunnel: <TUNNEL_ID>
credentials-file: /etc/cloudflared/credentials.json

ingress:
  - hostname: devspace.example.com
    service: http://localhost:3100
    originRequest:
      noTLSVerify: false
      connectTimeout: 30s
      tlsTimeout: 10s

  - service: http_status:404
```

### 3. 启动 Tunnel

```bash
cloudflared tunnel run --config tunnel.yaml devspace-tunnel
# 或后台运行
nohup cloudflared tunnel run --config tunnel.yaml devspace-tunnel > /var/log/cloudflared-devspace.log 2>&1 &
```

### 4. Endpoint 信息

| 项目 | 值 |
|------|-----|
| 内部地址 | `http://localhost:3100` |
| 公网地址 | `https://devspace.example.com` |
| MCP Endpoint | `https://devspace.example.com/mcp` |

---

## Agent Orchestrator + Supergateway 部署

### 1. 环境准备

```bash
# 安装 Node.js (Supergateway 依赖)
curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
apt-get install -y nodejs

# 安装 Claude Code CLI
npm install -g @anthropic-ai/claude-code
```

### 2. 启动 Supergateway

```bash
# 安装 Supergateway
npm install -g supergateway

# 启动 Supergateway (Agent Orchestrator stdio → Streamable HTTP)
# ⚠️ 重要：必须设置 IS_SANDBOX=1 以支持 root 环境下运行 Claude Worker
IS_SANDBOX=1 supergateway \
  --port 3080 \
  --agent-command "agent-orchestrator" \
  --agent-args "start-worker --model claude-3-5-sonnet-20241022" \
  2>&1 | tee /var/log/supergateway.log &
```

### 3. 配置 Cloudflare Tunnel Ingress

```yaml
# tunnel-agent.yaml
tunnel: <TUNNEL_ID>
credentials-file: /etc/cloudflared/credentials-agent.json

ingress:
  - hostname: agent.example.com
    service: http://localhost:3080
    originRequest:
      noTLSVerify: false
      connectTimeout: 60s
      tlsTimeout: 30s
      httpHostHeader: localhost

  - service: http_status:404
```

### 4. 启动 Tunnel

```bash
cloudflared tunnel run --config tunnel-agent.yaml agent-tunnel
# 或后台运行
nohup cloudflared tunnel run --config tunnel-agent.yaml agent-tunnel > /var/log/cloudflared-agent.log 2>&1 &
```

### 5. Endpoint 信息

| 项目 | 值 |
|------|-----|
| 内部地址 | `http://localhost:3080` |
| 公网地址 | `https://agent.example.com` |
| MCP Endpoint | `https://agent.example.com/mcp` |

---

## Cloudflare Tunnel 配置

### 完整 Ingress 示例

```yaml
# cloudflared-tunnel.yaml
tunnel: <TUNNEL_ID>
credentials-file: /etc/cloudflared/credentials.json

ingress:
  # DevSpace - 原生 HTTP MCP
  - hostname: devspace.example.com
    service: http://localhost:3100
    path: /mcp
    originRequest:
      noTLSVerify: false
      connectTimeout: 30s
      tlsTimeout: 10s

  # Agent Orchestrator - Streamable HTTP via Supergateway
  - hostname: agent.example.com
    service: http://localhost:3080
    originRequest:
      noTLSVerify: false
      connectTimeout: 60s
      tlsTimeout: 30s
      httpHostHeader: localhost

  # 默认回退
  - service: http_status:404
```

### 验证 Tunnel 状态

```bash
# 查看活跃连接
cloudflared tunnel list

# 查看特定 Tunnel 日志
cloudflared tunnel logs <TUNNEL_ID> --last-5m

# 测试公网端点
curl -I https://devspace.example.com/mcp
curl -I https://agent.example.com/mcp
```

---

## ChatGPT MCP Endpoint 配置

### 配置界面

在 ChatGPT 设置 → Extensions → MCP Servers 中添加：

#### DevSpace 配置

```json
{
  "name": "DevSpace",
  "mcp_servers": [
    {
      "type": "http",
      "url": "https://devspace.example.com/mcp"
    }
  ]
}
```

#### Agent Orchestrator 配置

```json
{
  "name": "Agent Orchestrator",
  "mcp_servers": [
    {
      "type": "http",
      "url": "https://agent.example.com/mcp"
    }
  ]
}
```

### 两种配置同时使用

在 ChatGPT 中可以根据任务类型选择使用哪个 MCP：

- **DevSpace**：快速代码审查、小修改、git 操作
- **Agent Orchestrator**：长时间构建、远程部署、持续监控

---

## 测试与验收

### DevSpace 测试

#### 测试用提示词

```
请帮我完成以下任务：
1. 列出 /home/user/project/my-project 目录结构
2. 查看最近的 git commit 历史（3条）
3. 检查 package.json 中的依赖版本

完成后请汇报结果。
```

#### 验收标准

| 检查项 | 预期结果 |
|--------|----------|
| 目录列表 | 返回完整的项目目录结构 |
| Git 历史 | 显示最近3条 commit，含 hash、author、message |
| 依赖检查 | 成功读取并解析 package.json |

### Agent Orchestrator + Claude Worker 测试

#### 测试用提示词

```
请完成以下验证任务：
1. 执行 pwd 命令，确认工作目录
2. 执行 hostname 命令，确认主机名
3. 执行 git status，确认当前仓库状态
4. 如果环境允许，尝试执行 ssh -V 确认 SSH 客户端可用

完成后报告每个命令的输出和执行状态。
```

#### 验收标准

| 检查项 | 预期结果 |
|--------|----------|
| pwd | 返回当前工作目录路径 |
| hostname | 返回主机名 |
| git status | 显示仓库状态（干净或有修改） |
| ssh -V | 显示 OpenSSH 版本信息 |

#### 通过 run_id 查询进度和结果

```bash
# 查看活跃 Worker
agent-orchestrator list-workers

# 查看特定 run 状态
agent-orchestrator get-run --run-id <RUN_ID>

# 查看 run 日志
agent-orchestrator logs --run-id <RUN_ID> --tail 50
```

---

## 工作流选择建议

### 选择 DevSpace 的场景

| 场景 | 说明 |
|------|------|
| 快速代码审查 | 读取源码、分析逻辑 |
| 小修改 | 修改配置文件、修复 bug |
| Git 操作 | 查看 diff、检查历史 |
| 短时构建 | 快速编译、运行单元测试 |
| 即时响应 | 需要秒级响应的交互 |

### 选择 Agent Orchestrator 的场景

| 场景 | 说明 |
|------|------|
| 长时间构建 | 完整项目编译、Docker 镜像构建 |
| 远程部署 | SSH 到服务器执行部署 |
| 网络测试 | iperf 带宽测试、持续 ping |
| 持续监控 | 长时间观察日志、监控指标 |
| 验收测试 | 执行完整测试套件并收集结果 |

### 协作模式示例

```
用户 → ChatGPT → DevSpace: "请审查这段代码实现"
   → Agent Orchestrator: "请在 <TEST_HOST> 上执行完整构建并部署"
   → ChatGPT: 汇总两个 Agent 的结果，形成完整报告
```

---

## 常见故障排查

### 1. DevSpace 连接超时

**现象**：`MCP connection timeout`

**可能原因**：
- DevSpace 服务未启动
- Cloudflare Tunnel 连接中断
- 端口被防火墙拦截

**排查步骤**：
```bash
# 检查 DevSpace 进程
ps aux | grep devspace

# 检查端口监听
netstat -tlnp | grep 3100

# 测试本地连接
curl -s http://localhost:3100/mcp/health

# 检查 Tunnel 状态
cloudflared tunnel list
```

**修复方法**：
```bash
# 重启 DevSpace
docker restart devspace
# 或
systemctl restart devspace

# 重启 Tunnel
cloudflared tunnel run --config tunnel.yaml devspace-tunnel
```

### 2. Agent Orchestrator Worker 启动失败

**现象**：`Worker failed to start` / `Permission denied`

**可能原因**：
- Claude Code 在 root 环境下被安全策略阻止
- IS_SANDBOX 环境变量未正确设置

**排查步骤**：
```bash
# 检查 Claude Code 版本
claude --version

# 检查环境变量
echo $IS_SANDBOX

# 查看 Supergateway 日志
tail -100 /var/log/supergateway.log
```

**修复方法**：
```bash
# 确保设置 IS_SANDBOX=1
export IS_SANDBOX=1

# 重启 Supergateway
pkill -f supergateway
IS_SANDBOX=1 supergateway --port 3080 --agent-command "agent-orchestrator" &
```

### 3. Claude Code root 权限问题

**现象**：`--dangerously-skip-permissions cannot be used with root/sudo privileges for security reasons`

**原因**：Claude Code 的安全策略禁止在 root 环境下使用跳过权限模式

**解决方案**：设置 `IS_SANDBOX=1` 环境变量

```bash
# 在启动 Supergateway 的环境中添加
export IS_SANDBOX=1

# 然后启动 Supergateway
IS_SANDBOX=1 supergateway --port 3080 ...
```

### 4. Supergateway Streamable HTTP 错误

**现象**：`Invalid MCP protocol` / `Streamable HTTP handshake failed`

**可能原因**：
- Supergateway 未正确启动
- Cloudflare Tunnel 未正确配置 httpHostHeader
- 协议版本不兼容

**排查步骤**：
```bash
# 测试 Supergateway 本地连接
curl -X POST http://localhost:3080/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"initialize","params":{},"id":1}'

# 检查 Cloudflare Tunnel 日志
cloudflared tunnel logs <TUNNEL_ID> --last-2m
```

### 5. Cloudflare Tunnel 连接不稳定

**现象**：MCP 连接间歇性断开

**可能原因**：
- 网络抖动
- Tunnel 配置的 timeout 过短
- Cloudflare 侧连接限制

**优化配置**：
```yaml
originRequest:
  connectTimeout: 60s
  tlsTimeout: 30s
  httpHostHeader: localhost
  # 增加保活
  keepAliveTimeout: 90s
```

---

## 安全提示

### ⚠️ IS_SANDBOX=1 安全风险

**重要说明**：本文档记录的 `IS_SANDBOX=1` 环境变量解决方案会改变 Claude Code 的安全判断逻辑。

#### 受影响的安全检查

- 允许 root 环境下使用 `--permission-mode bypassPermissions`
- 禁用部分文件系统访问限制
- 允许执行更多高权限操作

#### 适用场景

`IS_SANDBOX=1` **仅适用于**：

1. 用户明确知晓并认可的受控环境
2. 隔离的测试/开发环境
3. 不包含敏感数据或关键系统的沙箱

#### 不适用场景

- 生产环境
- 包含敏感客户数据的系统
- 多租户共享环境
- 关键基础设施

#### 替代方案

如有可能，建议：

1. **使用非 root 用户运行 Claude Worker**
   ```bash
   # 创建专用用户
   useradd -m -s /bin/bash claude-worker
   
   # 以该用户运行
   su - claude-worker -c "agent-orchestrator start-worker"
   ```

2. **使用容器隔离**
   ```bash
   docker run --rm -it \
     --user claude-worker \
     -v /home/user/project/my-project:/workspace \
     agent-orchestrator:latest
   ```

3. **配置细粒度权限**
   - 限制 Worker 可访问的目录范围
   - 使用 AppArmor/SELinux 强制访问控制

#### 安全最佳实践

- **不要**在生产环境使用 `IS_SANDBOX=1`
- **不要**将包含敏感数据的目录映射给启用沙箱的 Worker
- **始终**记录和审计 Worker 执行的操作
- **定期**审查访问日志和操作记录

---

## 附录：完整启动脚本示例

### DevSpace 一键启动

```bash
#!/bin/bash
# devspace-start.sh

set -e

PROJECT_PATH="/home/user/project/my-project"
TUNNEL_CONFIG="/etc/cloudflared/devspace-tunnel.yaml"

echo "Starting DevSpace..."

# 启动 DevSpace
docker run -d \
  --name devspace \
  --restart unless-stopped \
  -p 3100:3100 \
  -v ${PROJECT_PATH}:/workspace \
  -e PROJECT_PATH=/workspace \
  devspace:latest

# 等待就绪
sleep 3

# 启动 Tunnel
cloudflared tunnel run --config ${TUNNEL_CONFIG} devspace-tunnel &

echo "DevSpace started: https://devspace.example.com"
```

### Agent Orchestrator + Supergateway 一键启动

```bash
#!/bin/bash
# agent-orchestrator-start.sh

set -e

export IS_SANDBOX=1
export AGENT_WORKER_MODEL="claude-3-5-sonnet-20241022"
TUNNEL_CONFIG="/etc/cloudflared/agent-tunnel.yaml"

echo "Starting Agent Orchestrator with Supergateway..."

# 启动 Supergateway
nohup supergateway \
  --port 3080 \
  --agent-command "agent-orchestrator" \
  --agent-args "start-worker --model ${AGENT_WORKER_MODEL}" \
  > /var/log/supergateway.log 2>&1 &

SG_PID=$!
echo "Supergateway PID: $SG_PID"

# 等待就绪
sleep 5

# 启动 Tunnel
cloudflared tunnel run --config ${TUNNEL_CONFIG} agent-tunnel &

echo "Agent Orchestrator started: https://agent.example.com"
echo "Worker logs: /var/log/supergateway.log"
```

---

## 版本信息

| 项目 | 版本 | 说明 |
|------|------|------|
| Agent Orchestrator | 0.3.0 | 支持持久 Worker |
| Supergateway | latest | stdio → Streamable HTTP |
| Claude Code CLI | 最新稳定版 | - |
| Cloudflare Tunnel | 最新稳定版 | - |

---

## 参考链接

- [Agent Orchestrator GitHub](https://github.com/your-org/agent-orchestrator)
- [Supergateway](https://github.com/your-org/supergateway)
- [Cloudflare Tunnel 文档](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/)
- [MCP 协议规范](https://modelcontextprotocol.io)