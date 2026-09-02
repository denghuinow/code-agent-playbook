# DevSpace + Agent Orchestrator 通过 Supergateway 和 Cloudflare Tunnel 接入 ChatGPT

## 1. 目标

在一台 Linux 开发服务器上，将以下两类能力同时接入 ChatGPT：

* **DevSpace**

  * ChatGPT 直接读取、搜索、修改项目源码
  * 执行短时 shell、git、构建和快速测试

* **Agent Orchestrator**

  * ChatGPT 启动 Claude Code / Codex Worker
  * Worker 独立执行耗时较长的构建、SSH、iperf、监控和验收任务
  * ChatGPT 可以通过 `run_id` 查询进度、获取结果和继续同一个 Agent 会话

推荐架构：

```text
                         ┌──────────────────────┐
                         │       ChatGPT        │
                         └──────────┬───────────┘
                                    │ HTTPS MCP
                    ┌───────────────┴────────────────┐
                    │                                │
                    ▼                                ▼
       devspace.example.com              agent.example.com
                    │                                │
                    └────── Cloudflare Tunnel ──────┘
                    │                                │
                    ▼                                ▼
          127.0.0.1:7676                   127.0.0.1:8080
                    │                                │
                DevSpace                       Supergateway
                                                     │
                                                   stdio
                                                     │
                                                     ▼
                                             Agent Orchestrator
                                                     │
                                           persistent daemon
                                             ┌───────┴───────┐
                                             ▼               ▼
                                        Claude Code        Codex
```

**DevSpace 本身已经提供 HTTP MCP，因此不需要经过 Supergateway。**

DevSpace 默认 MCP 地址为：

```text
http://127.0.0.1:7676/mcp
```

Agent Orchestrator 0.3.0 默认提供 stdio MCP，因此需要 Supergateway 将 stdio 转换为 Streamable HTTP。

---

## 2. 环境要求

本次验证环境：

```text
Linux: Debian
Node.js: 22.x
DevSpace: @waishnav/devspace
Agent Orchestrator: @ralphkrauss/agent-orchestrator@0.3.0
Claude Code: 2.1.246
Cloudflare Tunnel: cloudflared
```

DevSpace 官方推荐使用 Node 22，并支持通过公网 HTTPS Tunnel 暴露本地 MCP。

---

# 3. 部署 DevSpace

## 3.1 初始化

安装或直接使用 npx：

```bash
npx @waishnav/devspace init
```

允许访问的项目目录例如：

```text
/root/project
```

项目：

```text
/root/project/ESP32-S3-UACNet
```

DevSpace 默认监听：

```text
http://127.0.0.1:7676/mcp
```

公网地址配置时填写 **origin，不要带 `/mcp`**：

```text
https://devspace.example.com
```

而 ChatGPT 中实际配置的 MCP Endpoint 才是：

```text
https://devspace.example.com/mcp
```

DevSpace 使用 Owner Password 对首次 MCP 客户端连接进行授权，密码保存在：

```text
~/.devspace/auth.json
```

## 3.2 systemd 示例

```ini
[Unit]
Description=DevSpace MCP Server
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
WorkingDirectory=/root/project

Environment="HOME=/root"
Environment="DEVSPACE_TOOL_MODE=full"
Environment="DEVSPACE_ALLOWED_ROOTS=/root/project"
Environment="DEVSPACE_PUBLIC_BASE_URL=https://devspace.example.com"
Environment="DEVSPACE_TRUST_PROXY=1"
Environment="PATH=/root/.nvm/versions/node/v22.23.2/bin:/root/.local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"

ExecStart=/root/.nvm/versions/node/v22.23.2/bin/devspace serve

Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

启用：

```bash
systemctl daemon-reload
systemctl enable --now devspace
systemctl status devspace
```

---

# 4. 部署 Agent Orchestrator

先确认 Claude / Codex：

```bash
claude --version
codex --version
```

运行诊断：

```bash
npx -y @ralphkrauss/agent-orchestrator@0.3.0 doctor
```

Agent Orchestrator 的设计是：

```text
MCP frontend
    ↓
persistent daemon
    ↓
Claude / Codex Worker
```

daemon 负责 Worker 进程、日志、超时、run metadata 和 Agent session reuse，因此 Worker 不依赖当前 ChatGPT 回合持续存在。

---

# 5. 使用 Supergateway 转换 Agent Orchestrator

Supergateway 支持：

```text
stdio
  ↓
Streamable HTTP
```

并支持 stateful 模式和 `/mcp` Endpoint。

测试命令：

```bash
npx -y supergateway \
  --stdio "npx -y @ralphkrauss/agent-orchestrator@0.3.0" \
  --outputTransport streamableHttp \
  --stateful \
  --port 8080 \
  --streamableHttpPath /mcp
```

本地 MCP Endpoint：

```text
http://127.0.0.1:8080/mcp
```

---

# 6. root 运行 Claude Code 的特殊配置

如果 Agent Orchestrator 以 `root` 运行，并启动：

```text
claude --permission-mode bypassPermissions
```

Claude Code 默认会拒绝启动，并出现：

```text
--dangerously-skip-permissions cannot be used with root/sudo privileges for security reasons
```

本环境已经实际验证，可在启动 Agent Orchestrator 的环境中加入：

```text
IS_SANDBOX=1
```

例如：

```bash
IS_SANDBOX=1 npx -y supergateway \
  --stdio "npx -y @ralphkrauss/agent-orchestrator@0.3.0" \
  --outputTransport streamableHttp \
  --stateful \
  --port 8080 \
  --streamableHttpPath /mcp
```

验证后 Claude 可以以：

```text
root
+
IS_SANDBOX=1
+
--permission-mode bypassPermissions
```

正常启动。

> 注意：这是一个安全敏感配置。它意味着允许 root Claude Worker 以免审批模式执行 shell、修改文件和访问网络，只应在完全可信、隔离且明确授权的开发服务器上使用。

---

# 7. Agent Orchestrator systemd 示例

建议让 systemd 同时管理 Supergateway 和 Agent Orchestrator：

```ini
[Unit]
Description=Agent Orchestrator MCP Gateway
After=network-online.target
Wants=network-online.target

[Service]
Type=simple

User=root
WorkingDirectory=/root/project/ESP32-S3-UACNet

Environment="HOME=/root"
Environment="IS_SANDBOX=1"
Environment="PATH=/root/.local/bin:/root/.nvm/versions/node/v22.23.2/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"

ExecStart=/root/.nvm/versions/node/v22.23.2/bin/npx -y supergateway --stdio "/root/.nvm/versions/node/v22.23.2/bin/npx -y @ralphkrauss/agent-orchestrator@0.3.0" --outputTransport streamableHttp --stateful --port 8080 --streamableHttpPath /mcp

Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

启用：

```bash
systemctl daemon-reload
systemctl enable --now agent-orchestrator-mcp
systemctl status agent-orchestrator-mcp
```

检查监听端口：

```bash
ss -lntp | grep -E '7676|8080'
```

应该看到：

```text
127.0.0.1:7676   DevSpace
127.0.0.1:8080   Supergateway
```

---

# 8. 配置 Cloudflare Tunnel

一个 Cloudflare Tunnel 可以同时发布多个本地 HTTP 服务，例如：

```yaml
tunnel: <TUNNEL-ID>
credentials-file: /root/.cloudflared/<TUNNEL-ID>.json

ingress:
  - hostname: devspace.example.com
    service: http://127.0.0.1:7676

  - hostname: agent.example.com
    service: http://127.0.0.1:8080

  - service: http_status:404
```

Cloudflare 官方要求 ingress 最后包含 catch-all rule。

验证：

```bash
cloudflared tunnel ingress validate
```

然后：

```bash
systemctl restart cloudflared
systemctl status cloudflared
```

Cloudflare Tunnel 本身通过出站连接建立隧道，因此开发服务器无需开放公网入站端口。

---

# 9. 在 ChatGPT 中添加两个 MCP

分别建立两个自定义 MCP App。

## DevSpace

Endpoint：

```text
https://devspace.example.com/mcp
```

首次授权时完成 DevSpace Owner Password approval。

## Agent Orchestrator

Endpoint：

```text
https://agent.example.com/mcp
```

点击：

```text
Scan Tools
```

正常情况下应该能够看到类似：

```text
start_run
send_followup
get_run_status
get_run_progress
get_run_result
wait_for_run
cancel_run
start_orchestration_run
...
```

ChatGPT 当前自定义 MCP App 配置流程支持填写远程 MCP Endpoint、选择适用的认证机制并扫描工具。

ChatGPT 不能直接连接 localhost MCP，因此必须通过公网 Remote MCP Endpoint、Secure MCP Tunnel 或其他可访问的 HTTPS MCP 入口。这里使用的是 Cloudflare Tunnel。

---

# 10. 测试 DevSpace

在 ChatGPT 中启用 DevSpace，然后发送：

```text
使用 DevSpace。

工作区：
/root/project/ESP32-S3-UACNet

请：
1. 打开工作区
2. 读取 AGENTS.md
3. 执行 pwd
4. 执行 hostname
5. 执行 git status --short

不要修改文件。
```

通过标准：

```text
ChatGPT 能成功 open_workspace
能够读取项目文件
pwd = /root/project/ESP32-S3-UACNet
hostname 正确
git status 正常返回
```

然后可以进一步测试 DevSpace 的 `edit/write` 功能。

---

# 11. 测试 Agent Orchestrator + Claude

发送：

```text
使用 Agent Orchestrator 启动 Claude trusted Worker。

工作区：
/root/project/ESP32-S3-UACNet

不要修改任何文件。

实际执行：
1. pwd
2. hostname
3. git status --short
4. ssh -o BatchMode=yes -o ConnectTimeout=10 172.16.15.80 hostname

返回 run_id 和实际结果。
```

通过标准：

```text
backend = claude
status = completed
能够生成 Claude session_id
pwd 成功
hostname 成功
git status 成功
SSH 成功
```

本环境实测结果：

```text
run_id:
01M1FYYC9HVKS4K70B238TSWBB

Claude session_id:
d86d3d9b-2b1c-4286-96ea-f3515456ea4a
```

SSH：

```text
ssh 172.16.15.80 hostname
→ softAP-client
```

全部退出状态为 `0`。

这同时证明：

```text
ChatGPT
→ Cloudflare
→ Supergateway
→ Agent Orchestrator
→ Claude Code
→ shell / SSH
```

链路已经完整工作。

---

# 12. 测试持久 Worker

这是 Agent Orchestrator 最重要的测试。

发送：

```text
使用 Agent Orchestrator 启动 Claude trusted Worker。

执行约 2 分钟的只读测试：

连续 6 次执行：

date '+%F %T %z'
hostname

两次之间 sleep 20。

不要修改文件。

启动后立即告诉我 run_id，让 Worker 自己继续运行。
```

获得：

```text
run_id = xxxxx
```

之后可以离开当前执行过程，稍后再次发送：

```text
检查 run_id xxxxx 的当前进度。
```

最终发送：

```text
获取 run_id xxxxx 的最终结果。
```

如果能够获得完整的 6 次采样，则说明：

```text
ChatGPT 当前回合结束
        ↓
Agent Orchestrator daemon 仍运行
        ↓
Claude Worker 继续工作
        ↓
之后可以通过 run_id 获取结果
```

持久 Worker 功能验证通过。

---

# 13. 推荐的实际工作方式

最终建议职责分工如下：

```text
ChatGPT
 │
 ├── DevSpace
 │     ├── Read
 │     ├── Grep
 │     ├── Edit / Write
 │     ├── git diff
 │     └── 短时 build / test
 │
 └── Agent Orchestrator
       └── Claude trusted Worker
             ├── 长时间构建
             ├── SSH
             ├── OTA
             ├── iperf
             ├── UART / carrier monitor
             └── 长时间验收
```

推荐提示词：

```text
使用 DevSpace + Agent Orchestrator 完成工作。

工作区：
/root/project/ESP32-S3-UACNet

使用 DevSpace 读取和修改源码，并执行短时构建和验证。

耗时较长的测试、SSH、iperf、持续监控和验收，
使用 Agent Orchestrator 启动 Claude trusted Worker 执行。

保留 run_id，并根据 Worker 测试结果继续通过 DevSpace 修改代码，
再让 Claude Worker 复测，直到完成。

不要只给操作建议；必须实际执行。
```

---

# 14. 常见故障

## DevSpace `/` 返回 404

ChatGPT Endpoint 必须是：

```text
https://devspace.example.com/mcp
```

不是：

```text
https://devspace.example.com
```

而：

```text
DEVSPACE_PUBLIC_BASE_URL
```

反而必须只填写 origin，不能包含 `/mcp`。

## Cloudflare 返回 502

检查：

```bash
curl http://127.0.0.1:7676/mcp
curl http://127.0.0.1:8080/mcp
```

以及：

```bash
systemctl status devspace
systemctl status agent-orchestrator-mcp
systemctl status cloudflared
```

## Claude Worker 启动后立即 exit 1

检查 Agent Orchestrator run 的：

```text
stderr.log
```

如果看到：

```text
--dangerously-skip-permissions cannot be used with root/sudo privileges
```

确认启动 Agent Orchestrator/Supergateway 的 systemd service 中存在：

```ini
Environment="IS_SANDBOX=1"
```

然后：

```bash
systemctl daemon-reload
systemctl restart agent-orchestrator-mcp
```

## Agent Orchestrator 显示 auth_unknown

`auth_unknown` 不一定表示无法使用。

最可靠的验证方式是实际启动：

```text
start_run
```

如果能够建立 Claude/Codex session 并产生模型输出，则认证实际可用。

---

# 15. 最终验收标准

全部满足以下条件即可认为接入完成：

```text
[ ] DevSpace MCP 能被 ChatGPT 扫描
[ ] ChatGPT 能打开指定 workspace
[ ] ChatGPT 能通过 DevSpace 读取源码
[ ] ChatGPT 能通过 DevSpace 修改源码
[ ] ChatGPT 能通过 DevSpace 执行短时 shell

[ ] Agent Orchestrator MCP 能被 ChatGPT 扫描
[ ] ChatGPT 能成功 start_run
[ ] Claude trusted Worker 能正常启动
[ ] root + IS_SANDBOX=1 配置生效
[ ] Claude 能执行 shell
[ ] Claude 能 SSH 到测试机
[ ] 能获得 run_id
[ ] 能通过 run_id 查询进度
[ ] 能通过 run_id 获取最终结果
[ ] 长任务不会因为当前 ChatGPT 回合结束而被取消
```

全部通过后，推荐正式工作模式为：

```text
DevSpace = ChatGPT 的代码操作层
Agent Orchestrator = ChatGPT 的持久任务执行层
Claude Code = 长任务 Worker
Supergateway = Agent Orchestrator 的 stdio → HTTP MCP 网关
Cloudflare Tunnel = 两个 MCP 的公网 HTTPS 入口
```
