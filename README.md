# dao · 大道至简 · 太上 下知有之

> GitHub Issues Comments = 无感传输层

PowerShell窗口不关闭 = 一直可操控。GitHub不知道你在用它。太上，下知有之。

## 架构

```
dao-call ──POST──> GitHub Issue Comment (dao-cmd:base64)
                         │
agent.ps1 ──GET──poll────┘  ← If-Modified-Since: 304=free
                         │
agent.ps1 ──POST──> GitHub Issue Comment (dao-result:base64)
                         │
dao-call ──GET──poll────┘
```

**核心**: GitHub Issues Comments API = 邮箱。命令和结果都是base64 JSON的comment。

**304 Not Modified不计配额**: agent轮询用If-Modified-Since，无新命令时返回304（免费），只有新命令到达时才消耗配额。60/hr配额可持续运行。

## 一行启动

```powershell
# Agent (被控端) — PowerShell窗口保持打开
$env:DAO_TOKEN='ghp_xxx'; irm https://raw.githubusercontent.com/zhouyoukang1234-spec/devin-remote/main/agent.ps1 | iex

# 或带参数
.\agent.ps1 -Token ghp_xxx -Poll 10
```

```bash
# Commander (控制端) — PowerShell
$env:DAO_TOKEN='ghp_xxx'; . .\dao-call.ps1
dao 141 hostname           # shell命令
dao-shot 141               # 截屏
dao-sys 179                # 系统信息

# Commander — Bash
DAO_TOKEN=ghp_xxx ./dao-exec.sh -a 141 hostname
DAO_TOKEN=ghp_xxx ./dao-exec.sh -a 179 -t screenshot
DAO_TOKEN=ghp_xxx ./dao-exec.sh --agents
```

## 环境变量

| 变量 | 说明 | 默认 |
|------|------|------|
| `DAO_TOKEN` | GitHub PAT (classic, repo scope) | 交互输入 |
| `DAO_REPO` | owner/repo | `zhouyoukang1234-spec/devin-remote` |
| `DAO_POLL` | agent轮询间隔(秒) | `10` |
| `DAO_PROXY` | HTTP代理 | 无 |
| `DAO_TIMEOUT` | dao-call等结果超时(秒) | `120` |

## 命令

| 命令 | 说明 | 示例 |
|------|------|------|
| `dao <agent> <cmd>` | Shell命令 | `dao 141 hostname` |
| `dao-shot <agent>` | 截屏 | `dao-shot 179` |
| `dao-sys <agent>` | 系统信息 | `dao-sys 141` |
| `dao-ps <agent>` | 进程列表 | `dao-ps 141 chrome` |
| `dao-kill <agent> -ProcessId N` | 杀进程 | `dao-kill 141 -ProcessId 1234` |
| `dao-ls <agent> [path]` | 列目录 | `dao-ls 141 C:\Users` |
| `dao-get <agent> <path>` | 读文件 | `dao-get 141 C:\Windows\win.ini` |
| `dao-put <agent> <local> <remote>` | 写文件 | `dao-put 141 file.txt C:\file.txt` |
| `dao-reg <agent> <path>` | 读注册表 | `dao-reg 141 HKLM:\SOFTWARE` |
| `dao-svc <agent>` | 服务列表 | `dao-svc 141` |
| `dao-net <agent>` | 网络信息 | `dao-net 141` |
| `dao-env <agent>` | 环境变量 | `dao-env 141` |
| `dao-apps <agent>` | 已装应用 | `dao-apps 141` |

## 别名

| 别名 | 实际Agent ID |
|------|-------------|
| `141` / `desktop` | `DESKTOP-MASTER` |
| `179` / `laptop` | `ZHOUMAC` |

## Mailbox机制

每个agent对应一个GitHub Issue（标题 `mailbox-<COMPUTERNAME>`，标签 `dao-mailbox`）。

- **发现**: `GET /issues?labels=dao-mailbox` — 1次API调用找到所有mailbox
- **创建**: 不存在则自动创建，带 `dao-mailbox` 标签
- **缓存**: 本地 `~/.dao-mailbox` 缓存issue编号

## Rate Limit策略

GitHub API配额有限（classic PAT 5000/hr，free账号可能更低）。

**If-Modified-Since**: agent轮询时带 `If-Modified-Since` 头。无新命令时GitHub返回 `304 Not Modified`，**304不计配额**。只有新命令到达时才消耗1次配额。

**403处理**: 遇到403时读取 `X-RateLimit-Reset` 头，精确等待到reset时间。

**Poll间隔**: 默认10秒。即使60/hr配额，304免费机制下可持续运行。

## 设计哲学

> 大道至简，无为而无以为。

- GitHub不知道你在用它（太上，下知有之）
- 去掉一切不必要的层：ETag缓存、自适应轮询、心跳文件、rate追踪、proxy自动发现、安装/卸载
- 保留If-Modified-Since（304=free，道法自然）
- 用labels查询mailbox（1次API代替20次扫描）
- PowerShell窗口不关 = 一直可操控
