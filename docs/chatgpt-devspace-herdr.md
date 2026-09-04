# ChatGPT + DevSpace + Herdr + Claude 持久化工作流

## 1. 目标

在 Linux 开发机上建立以下调用链：

```text
ChatGPT
  ↓ HTTPS MCP
DevSpace
  ↓ Herdr CLI / local socket
Herdr headless server
  ↓ managed pane
Claude Code
```

实现：

- ChatGPT 通过 DevSpace 读写源码、执行命令。
- DevSpace 自身运行在 Herdr 管理的 pane 中，获得 `HERDR_ENV=1`。
- ChatGPT 可以通过 Herdr 创建 Claude pane、提交长任务并间隔查询状态。
- Herdr 和 DevSpace 由 systemd 开机启动，服务器重启后无需手工打开 Herdr。
- Claude 长任务不依赖单次 MCP 请求持续连接。

## 2. 适用边界

本方案适合单机 Linux 开发环境，Herdr、DevSpace 和 Claude Code 运行在同一用户下。

本次验证环境：

```text
Linux: Debian
Node.js: 22.x
Herdr: 0.8.2
Claude Code: 2.1.246
DevSpace: @waishnav/devspace
```

示例约定：

```text
运行用户: root
项目根目录: /srv/projects
DevSpace: /root/.nvm/versions/node/v22.23.2/bin/devspace
Herdr: /root/.local/bin/herdr
Herdr socket: /root/.config/herdr/herdr.sock
DevSpace 公网地址: https://devspace.example.com
```

实际部署时应替换路径、用户和域名。

## 3. 为什么不能独立启动 DevSpace

Herdr 的 Agent skill 要求调用方位于 Herdr 管理的 pane 中：

```bash
test "${HERDR_ENV:-}" = 1
```

Herdr 会为 pane 内进程注入：

```text
HERDR_ENV=1
HERDR_SOCKET_PATH=...
HERDR_WORKSPACE_ID=...
HERDR_TAB_ID=...
HERDR_PANE_ID=...
```

如果直接由普通 `devspace.service` 启动 DevSpace，DevSpace 不属于任何 Herdr pane，因而不应直接控制当前 Herdr 会话。

另一个常见问题是 systemd 服务未设置 `HOME`：

```text
DevSpace 内 Herdr CLI 查找: /tmp/herdr/herdr.sock
实际 Herdr server socket: /root/.config/herdr/herdr.sock
```

因此本方案采用：

```text
systemd
├── herdr.service
└── devspace.service
      └── supervisor
            └── Herdr pane
                  └── devspace serve
```

## 4. 安装 Claude 集成

Herdr 可以通过终端画面检测 Claude 状态。安装官方集成后，还能记录原生 Claude session ID，并在支持的场景恢复会话。

```bash
herdr integration install claude
herdr integration status
```

## 5. 配置 Herdr headless 服务

本方案同时把当前 Herdr binary 自带的 Agent Skill 同步到 DevSpace 可以发现的全局 Skill 目录：

```text
Herdr --skill
      ↓ 自动同步
~/.agents/skills/herdr/SKILL.md
      ↓ DevSpace 自动发现
DevSpace MCP
      ↓
ChatGPT
```

这样 ChatGPT 可以通过 DevSpace 获得与当前 Herdr 版本匹配的使用规则，不需要在每次任务中重新执行 `herdr --skill`。

创建 `/etc/systemd/system/herdr.service`：

```ini
[Unit]
Description=Herdr Headless Terminal Server
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=root
WorkingDirectory=/root
Environment=HOME=/root
Environment=XDG_CONFIG_HOME=/root/.config
Environment=HERDR_CONFIG_PATH=/root/.config/herdr/config.toml
Environment=HERDR_SOCKET_PATH=/root/.config/herdr/herdr.sock
Environment=PATH=/root/.local/bin:/root/.nvm/versions/node/v22.23.2/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
ExecStartPre=/usr/bin/install -d -m 0755 /root/.agents/skills/herdr
ExecStartPre=/bin/sh -c '/root/.local/bin/herdr --skill > /root/.agents/skills/herdr/SKILL.md.tmp && mv /root/.agents/skills/herdr/SKILL.md.tmp /root/.agents/skills/herdr/SKILL.md'
ExecStart=/root/.local/bin/herdr server
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

`herdr server` 只运行后台服务，不需要保持 Herdr TUI 窗口开启。

两个 `ExecStartPre` 会在每次 Herdr 服务启动前：

- 创建 `~/.agents/skills/herdr`。
- 执行当前 binary 的 `herdr --skill`。
- 原子更新 `SKILL.md`，避免写入过程中留下半文件。
- Herdr 升级后，只要重启 `herdr.service`，Skill 就会随当前版本一起刷新。

## 6. 创建 DevSpace pane 监督器

创建 `/usr/local/libexec/devspace-herdr-supervisor`：

```bash
#!/usr/bin/env bash
set -u

HERDR_BIN=/root/.local/bin/herdr
DEVSPACE_BIN=/root/.nvm/versions/node/v22.23.2/bin/devspace
PROJECT_ROOT=/srv/projects
PUBLIC_BASE_URL=https://devspace.example.com
STATE_DIR=/var/lib/devspace-herdr
PANE_STATE=${STATE_DIR}/pane-id

export HOME=/root
export XDG_CONFIG_HOME=/root/.config
export HERDR_CONFIG_PATH=/root/.config/herdr/config.toml
export HERDR_SOCKET_PATH=/root/.config/herdr/herdr.sock
export PATH=/root/.nvm/versions/node/v22.23.2/bin:/root/.local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

mkdir -p "$STATE_DIR"
pane_id=""

log() {
    printf '%s devspace-herdr: %s\n' "$(date -Is)" "$*"
}

valid_pane() {
    [[ -n "$pane_id" ]] &&
        "$HERDR_BIN" pane get "$pane_id" >/dev/null 2>&1
}

ensure_pane() {
    if [[ -z "$pane_id" && -s "$PANE_STATE" ]]; then
        IFS= read -r pane_id < "$PANE_STATE"
    fi

    if valid_pane; then
        return 0
    fi

    pane_id=""
    local created
    created="$("$HERDR_BIN" workspace create \
        --cwd "$PROJECT_ROOT" \
        --label devspace-mcp \
        --env HOME=/root \
        --env XDG_CONFIG_HOME=/root/.config \
        --env PATH="$PATH" \
        --no-focus)" || return 1

    pane_id="$(printf '%s\n' "$created" |
        grep -o '"pane_id":"[^"]*"' |
        head -n 1 |
        cut -d '"' -f 4)"

    if [[ -z "$pane_id" ]]; then
        log "unable to parse pane id: $created"
        return 1
    fi

    printf '%s\n' "$pane_id" >"$PANE_STATE"
    log "created pane $pane_id"
}

stop_devspace() {
    trap - TERM INT
    if valid_pane; then
        "$HERDR_BIN" pane send-keys "$pane_id" ctrl+c             >/dev/null 2>&1 || true
    fi
    log "stopped"
    exit 0
}

trap stop_devspace TERM INT

until "$HERDR_BIN" status server >/dev/null 2>&1; do
    sleep 1
done

while true; do
    if ! "$HERDR_BIN" status server >/dev/null 2>&1; then
        pane_id=""
        sleep 1
        continue
    fi

    if ! ensure_pane; then
        sleep 2
        continue
    fi

    process_info="$("$HERDR_BIN" pane process-info \
        --pane "$pane_id" 2>/dev/null || true)"

    if ! printf '%s\n' "$process_info" | grep -q 'devspace.*serve'; then
        if printf '%s\n' "$process_info" |
            grep -q '"cmdline":"/bin/bash"'; then
            log "starting DevSpace in pane $pane_id"
            "$HERDR_BIN" pane run "$pane_id" \
                "cd $PROJECT_ROOT && env DEVSPACE_TRUST_PROXY=1 DEVSPACE_PUBLIC_BASE_URL=$PUBLIC_BASE_URL DEVSPACE_TOOL_MODE=full DEVSPACE_ALLOWED_ROOTS=$PROJECT_ROOT PATH=$PATH $DEVSPACE_BIN serve" \
                >/dev/null
        else
            log "pane $pane_id is busy; waiting"
        fi
    fi

    sleep 3 &
    wait $!
done
```

赋予执行权限：

```bash
chmod 0755 /usr/local/libexec/devspace-herdr-supervisor
```

监督器负责：

- 等待 Herdr server 可用。
- 创建或恢复专用 `devspace-mcp` workspace/pane。
- 在该 pane 中运行 `devspace serve`。
- DevSpace 退出后重新启动。
- 服务停止时向 DevSpace pane 发送 `Ctrl-C`。
- 通过持久化 pane ID 避免每次启动重复创建 workspace。

## 7. 修改 DevSpace systemd 服务

保留原有 `devspace.service`，增加 drop-in：

`/etc/systemd/system/devspace.service.d/herdr.conf`

```ini
[Unit]
Requires=herdr.service
After=herdr.service network-online.target
PartOf=herdr.service

[Service]
ExecStart=
ExecStart=/usr/local/libexec/devspace-herdr-supervisor
WorkingDirectory=/srv/projects
Environment=HOME=/root
Environment=XDG_CONFIG_HOME=/root/.config
Environment=HERDR_CONFIG_PATH=/root/.config/herdr/config.toml
Environment=HERDR_SOCKET_PATH=/root/.config/herdr/herdr.sock
Restart=always
RestartSec=3
KillMode=process
TimeoutStopSec=15
```

关键关系：

- `Requires`：DevSpace 依赖 Herdr。
- `After`：Herdr 先启动。
- `PartOf`：重启 Herdr 时联动重启 DevSpace。
- 清空原 `ExecStart`，改由监督器启动 pane 内的 DevSpace。

## 8. 启用服务

先检查配置：

```bash
bash -n /usr/local/libexec/devspace-herdr-supervisor
systemd-analyze verify \
  /etc/systemd/system/herdr.service \
  /etc/systemd/system/devspace.service
```

然后启用：

```bash
systemctl daemon-reload
systemctl enable herdr.service devspace.service
systemctl start herdr.service
systemctl restart devspace.service
```

如果已有手工启动的 Herdr server 占用同一个 socket，应先确认其中没有重要任务，再执行：

```bash
export HOME=/root
export HERDR_SOCKET_PATH=/root/.config/herdr/herdr.sock
herdr server stop
systemctl start herdr.service
systemctl restart devspace.service
```

风险：`herdr server stop` 会终止当前 Herdr server 管理的 pane 和进程。执行前必须确认没有正在运行的 Agent、构建或测试。

如果当前操作本身来自 DevSpace，重启会中断当前 MCP 请求。可以通过独立 SSH 终端执行，或使用 transient systemd unit 延迟切换：

```bash
systemd-run \
  --unit=herdr-devspace-migrate \
  --on-active=5s \
  /bin/bash -lc '
    systemctl stop devspace.service
    export HOME=/root
    export HERDR_SOCKET_PATH=/root/.config/herdr/herdr.sock
    herdr server stop || true
    systemctl start herdr.service
    systemctl start devspace.service
  '
```

## 9. 验证自动启动链

### 9.1 服务状态

```bash
systemctl is-enabled herdr.service devspace.service
systemctl is-active herdr.service devspace.service
```

预期：

```text
enabled
enabled
active
active
```

### 9.2 DevSpace 的 Herdr 环境

通过 ChatGPT 调用 DevSpace shell：

```bash
env | grep '^HERDR_' | sort
```

至少应出现：

```text
HERDR_ENV=1
HERDR_PANE_ID=...
HERDR_SOCKET_PATH=/root/.config/herdr/herdr.sock
HERDR_TAB_ID=...
HERDR_WORKSPACE_ID=...
```

### 9.3 Herdr Skill

确认当前 Herdr Skill 已同步：

```bash
test -s /root/.agents/skills/herdr/SKILL.md
head -n 20 /root/.agents/skills/herdr/SKILL.md
```

DevSpace 打开工作区后，应能从本地 Agent Skills 中发现该 Skill，使 ChatGPT 可以直接读取 Herdr 的当前版本使用规则。

### 9.4 进程归属

```bash
ps -eo pid,ppid,stat,args |
  grep -E '[h]erdr server|[d]evspace serve|[d]evspace-herdr-supervisor'
```

DevSpace 的父进程链应落在 Herdr server 管理的 pane shell 下，而不是直接由 systemd 启动。

### 9.5 重启恢复

```bash
systemctl restart herdr.service
```

等待数秒后重新连接 DevSpace，再检查：

```bash
systemctl is-active herdr.service devspace.service
test "${HERDR_ENV:-}" = 1
herdr status server
```

## 10. 调度 Claude 长任务

先确认当前 DevSpace 确实位于 Herdr 中：

```bash
test "${HERDR_ENV:-}" = 1
herdr pane current --current
```

查看布局并创建相邻 pane：

```bash
herdr pane layout --pane "$HERDR_PANE_ID"

split="$(herdr pane split \
  --current \
  --direction right \
  --cwd "$PWD" \
  --no-focus)"

printf '%s\n' "$split"
```

从响应的 `.result.pane.pane_id` 取得 pane ID，然后启动 Claude：

```bash
herdr agent start worker \
  --kind claude \
  --pane <pane-id> \
  --timeout 90000
```

提交任务时不要使用 `--wait` 占住一次 MCP 请求：

```bash
herdr agent prompt worker \
  "执行任务；不要提前返回，完成后输出明确的完成标记。"
```

之后由 ChatGPT 间隔查询：

```bash
herdr agent get worker
herdr agent read worker --source detection --lines 50
```

完成后获取输出：

```bash
herdr agent get worker
herdr agent read worker --source visible --lines 100
```

状态含义：

- `idle`：Agent 可接收任务，或任务已经完成并回到输入界面。
- `working`：Agent 正在执行。
- `blocked`：Agent 等待审批或用户输入。
- `unknown`：Herdr 无法可靠识别状态，不能视为完成。

## 11. 推荐提示词

Herdr 在这套方案中不是用来替代 DevSpace，而是作为 **持久化终端和 Agent 会话管理层**。

日常源码操作优先直接使用 DevSpace；当任务需要独立运行、耗时较长，或者需要启动 Claude 持续执行并在之后查询状态时，再使用 Herdr。

推荐分工：

```text
ChatGPT
 │
 ├── DevSpace
 │     ├── Read / Grep
 │     ├── Edit / Write
 │     ├── git diff
 │     └── 短时 shell / build / test
 │
 └── Herdr
       └── 独立持久化 pane
             └── Claude
                   ├── 长时间构建 / 测试
                   ├── SSH
                   ├── iperf
                   ├── 持续监控
                   └── 长时间验收
```

将下面内容中的工作区路径替换为实际项目路径后，可直接发送给 ChatGPT：

```text
使用 DevSpace + Herdr 完成工作。

工作区：
`/path/to/project`

工具使用规则：

- 使用 DevSpace：
  - 读取、搜索和修改源码
  - 查看 diff
  - 执行短时 shell、构建和快速验证
  - 能直接通过 DevSpace 完成的短任务，不要启动 Herdr Agent

- 使用 Herdr：
  - Herdr 用于管理独立、持久化的终端 pane 和 Claude Agent 会话
  - 当任务耗时较长、不适合占用一次 MCP 请求，或需要 SSH、iperf、持续监控、长时间测试和验收时，通过 Herdr 启动 Claude 执行
  - 使用 Herdr 时遵循 DevSpace 提供的 Herdr Skill；如果 `HERDR_ENV=1` 不成立，停止调用 Herdr 并说明原因
  - 长任务启动后不要阻塞等待，通过 Herdr 持续跟踪到完成或 blocked 状态
  - 若状态为 blocked，先读取阻塞原因，不要擅自批准高风险操作

根据 Claude 的测试或验收结果，继续通过 DevSpace 修改代码，需要时再通过 Herdr 让 Claude 复测，直到完成目标。

不要只给操作建议；必须实际执行。
```

核心判断原则：**短任务直接 DevSpace；需要独立持续运行的长任务，才使用 Herdr。**

## 12. 120 秒 PoC 实测

任务要求 Claude 执行：

```bash
start_epoch=$(date +%s)
start_iso=$(date -Is)
sleep 120
end_epoch=$(date +%s)
end_iso=$(date -Is)
elapsed=$((end_epoch - start_epoch))
```

ChatGPT 在任务期间多次调用 `herdr agent get`，状态变化为：

```text
提交后约 18 秒: working
提交后约 60 秒: working
提交后约 101 秒: working
任务结束后: idle
```

Claude 最终输出：

```text
POC_120S_COMPLETE
start_epoch=1788442109
end_epoch=1788442229
elapsed=120
start=2026-09-03T21:28:29+08:00
end=2026-09-03T21:30:29+08:00
```

结论：

- 长任务不会受单次 DevSpace MCP 请求时长限制。
- ChatGPT 可以在不同请求中持续查询同一 Claude Agent。
- Herdr 能正确识别 `working → idle` 状态变化。
- 任务实际耗时为 120 秒，没有提前结束。

## 13. root 用户限制

Claude Code 2.1.246 在 root 下会拒绝：

```bash
claude --dangerously-skip-permissions
```

错误：

```text
--dangerously-skip-permissions cannot be used with root/sudo privileges for security reasons
```

因此 root 环境不能直接宣称为 trusted Worker。本次 PoC 使用 Claude 正常 `auto mode`，计时任务没有触发审批。

推荐选择：

1. 使用普通服务账户运行 Herdr、DevSpace 和 Claude。
2. 使用 Claude 正常权限模式，并通过 Herdr 的 `blocked` 状态处理审批。
3. 只有当环境确实运行在受控沙箱中时，才使用与沙箱相关的绕过配置。

不要仅为跳过审批而伪造沙箱环境。

## 14. 常见问题

### Herdr 进程存在，但 CLI 报 server not running

现象：

```text
status: not running
socket: /tmp/herdr/herdr.sock
```

检查：

```bash
env | grep -E '^(HOME|XDG_CONFIG_HOME|HERDR_)='
ss -xlpn | grep herdr
```

修复：为 systemd 服务显式设置 `HOME`、`XDG_CONFIG_HOME`、`HERDR_CONFIG_PATH` 和 `HERDR_SOCKET_PATH`。

### `agent start` 超时

检查实际 pane：

```bash
herdr pane process-info --pane <pane-id>
herdr pane read <pane-id> --source detection --lines 80
```

不要把启动参数错误误判为 Herdr 检测故障。本次实测中，root 用户使用 `--dangerously-skip-permissions` 后 Claude 立即退出，表面上表现为启动超时。

### Agent 工作时无法读取较长历史

Herdr 可能返回：

```text
agent_not_idle
```

任务运行中使用：

```bash
herdr agent read worker --source detection --lines 50
```

任务回到 `idle` 后再读取更多历史。

## 15. 回滚

回滚会中断当前 DevSpace 与 Herdr 任务，应先确认没有正在执行的 Agent、构建或测试。

```bash
systemctl stop devspace.service herdr.service
systemctl disable herdr.service
rm -f /etc/systemd/system/devspace.service.d/herdr.conf
rm -f /etc/systemd/system/herdr.service
rm -f /usr/local/libexec/devspace-herdr-supervisor
rm -rf /root/.agents/skills/herdr
systemctl daemon-reload
systemctl restart devspace.service
```

回滚后，DevSpace 恢复使用原 `devspace.service` 中的直接 `ExecStart`，不再具备 `HERDR_ENV=1`，也不能按 Herdr Agent skill 控制当前会话。
