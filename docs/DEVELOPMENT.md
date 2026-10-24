# boardctl 开发文档

> 本文档是 boardctl 项目的**权威规格**。遵循"文档先行"原则（承接 vmpi 项目方法论）：任何已确认的需求，必须先写入本文档（需求记录章节追加条目），再进入开发。

| 版本 | 日期 | 变更 |
|---|---|---|
| v0.1 | 2025-10-24 | 初稿：定位（源自 vmpi REQ-0009 边界拆分）、需求记录（REQ-0001～0008）、架构、CLI 规格、阶段计划 Q0–Q5 |

---

## 1. 项目概述

### 1.1 定位

**boardctl = AI agent 的通用板卡操作客户端**：一个 Go 编写的 CLI，凭一个 endpoint（`host + ssh_port + user + 凭据`）为 AI agent（及人）提供对任意 arm64 Linux 开发板的远程操作能力。

- **统一 endpoint 模型**：虚拟板与真实板对 client 完全同构——都是"一台经网络可达的 arm64 Linux"。
- **双命令面**：使用面（板内操作，任意机器可执行）与管理面（板卡生命周期，经 `--via ssh` 代理到宿主机执行）。
- **面向 agent**：附 Agent Skills 规范的 skill；破坏性操作有防误伤设计；后续按需提供 MCP。

### 1.2 背景与来源

- 源起：[vmpi](https://github.com/kafkaliu/vmpi)（虚拟板模拟器，P0–P2 已交付）的边界讨论——"AI agent 经网络开发/测试/部署板子"的通用客户端层不适合放在模拟器仓库，独立成本项目（vmpi REQ-0009，2025-10-24 确认）。
- **与 vmpi 的关系**：
  - vmpi 回答"板子长在哪"（QEMU/WSL/usbipd 编排，运行在板卡宿主机）；
  - boardctl 回答"怎么用板子"（客户端，运行在 agent/开发机）；
  - boardctl 的管理面远程能力 = 经 ssh 在宿主机上调用 `vmpi` CLI（vmpi 是首个也是首个发布时唯一对接的宿主实现）。
- 首个用户场景：在开发机上由 AI agent 驱动，操作 vmpi 虚拟 Orange Pi Zero 3，完成 B（KVM→标准服务软件）的部署/验证工作流（含 NanoKVM-USB 直通板内的视频服务）。

### 1.3 非目标

- 不做虚拟化/模拟/镜像构建（vmpi 域）。
- 不做守护进程：保持"每次命令即时执行"模型；远程管理走 ssh 代理，不引入常驻服务。
- 不越界真板物理域：电源、烧录口、USB 插拔不软件化（`flash` 的 ssh+dd 远程写入为危险操作，条件评估，见 REQ-0005）。
- MCP server 为条件触发项（REQ-0007），不作为初始交付。

## 2. 需求记录（确认即固化）

> 规则：每次与需求方确认的结论，先追加到本表（含日期、编号、状态），再排入开发。状态：`已确认 / 开发中 / 已交付 / 已废弃`。

| 编号 | 日期 | 需求内容 | 状态 |
|---|---|---|---|
| REQ-0001 | 2025-10-24 | 定位：通用板卡客户端（虚拟/真实统一 endpoint 模型），供 AI agent 与人使用；源起 vmpi REQ-0009 边界拆分 | 已确认 |
| REQ-0002 | 2025-10-24 | 双命令面：使用面（ssh/deploy/console/status）任意机器可用；管理面（start/stop/reset/flash/usb/port）仅本地或经 `--via ssh://宿主机` 代理执行；远程调用管理面而无 via 时给出明确指引 | 已确认 |
| REQ-0003 | 2025-10-24 | 多板命名配置：`~/.boardctl/config.yaml` 以名字管理多个板子（虚拟/真板混排），含 host/ssh_port/user/key 或 password/console_port/via；`default` 指定默认板；优先级 flag > 环境变量 > 配置文件 > 默认值 | 已确认 |
| REQ-0004 | 2025-10-24 | 认证与引导：支持公钥与密码双认证；`boardctl auth` 首次引导——生成/复用本机密钥，用密码登录目标板注入公钥（等价 ssh-copy-id），之后免密 | 已确认 |
| REQ-0005 | 2025-10-24 | 真板支持与能力矩阵：ssh/deploy/status 完全一致；console 退化为 ssh（除非串口有网关）；电源/烧录口/USB 插拔为物理域不软件化；`flash`（ssh+dd 写 mmc）列为危险操作，Q4 后条件评估 | 已确认 |
| REQ-0006 | 2025-10-24 | 防误操作（agent 场景）：破坏性命令（deploy 覆盖、flash、stop、rm 类）默认需确认，`--yes` 跳过；命令输出对 agent 友好（可 machine-readable，如 `--json`） | 已确认 |
| REQ-0007 | 2025-10-24 | MCP server（条件触发）：当出现多客户端共享/工具级权限门控/非 shell host 需求时启动；实现为薄封装，工具标注 readOnly/destructive hint | 已确认（延后） |
| REQ-0008 | 2025-10-24 | 流程约定：文档先行 + 分级自动验收（selftest L0 静态 / L1 配置与连通 / L2 板上冒烟 / L3 真机条件执行），方法论承接 vmpi REQ-0005/0008 | 已确认 |

## 3. 架构

### 3.1 总体结构

```
AI agent（任意机器）
  └─ skill（知识层：何时/如何/禁忌，Agent Skills 规范）      ← Q3
       └─ boardctl client（Go CLI，无守护进程）
            ├─ 使用面：ssh / deploy / console / status
            │     └──> 统一 endpoint ──> 虚拟板（vmpi 宿主）或真实板
            └─ 管理面：start / stop / reset / flash / usb / port
                  ├─ 本地模式：直接执行（当前即 vmpi CLI 域）
                  └─ --via ssh://宿主机：在宿主机上代理执行 `vmpi <cmd>`（无守护进程）
```

### 3.2 配置模型（REQ-0003）

```yaml
# ~/.boardctl/config.yaml
default: vopi                    # 默认板名
boards:
  vopi:                          # vmpi 虚拟板（板卡宿主机 192.168.10.136）
    host: 192.168.10.136
    ssh_port: 2222
    user: orangepi
    key: ~/.boardctl/id_ed25519  # 二选一：key 或 password
    console_port: 23000          # vmpi port add 23000=23000 后可用
    via: orangepi@192.168.10.136 # 管理面 ssh 代理（宿主机 OpenSSH；可选）
  real-opi:                      # 真板示例
    host: 192.168.10.50
    ssh_port: 22
    user: orangepi
    password: orangepi
```

- 覆盖顺序：`--host/--user/--port/--key/--via` flags > `BOARDCTL_<...>` 环境变量 > 配置文件 > 默认值。
- 凭据只存本机配置；密码认证仅建议用于首次 `auth` 引导。

### 3.3 命令面划分

| 命令 | 面向 | 虚拟板 | 真板 | 说明 |
|---|---|---|---|---|
| `boardctl doctor` | — | ✅ | ✅ | 配置校验、ssh 连通性、认证方式探测 |
| `boardctl auth` | 使用面 | ✅ | ✅ | 首次引导注入公钥 |
| `boardctl ssh [cmd]` | 使用面 | ✅ | ✅ | 登录/执行命令 |
| `boardctl deploy <path> --svc n` | 使用面 | ✅ | ✅ | 本机交叉编译（GOOS=linux GOARCH=arm64，CGO=0）→ SFTP → systemd |
| `boardctl console` | 使用面 | ✅（TCP） | ⚠️ 退化为 ssh | 虚拟板连 `host:console_port`；真板提示无串口网关 |
| `boardctl status` | 使用面 | ✅ | ✅ | 板内视角：uptime/内存/存储/指定服务状态 |
| `boardctl start/stop/reset/flash/usb/port` | 管理面 | ✅（本地或 via） | ❌ 物理域 | via = `ssh via "vmpi <同参>"`；真板调用给出物理操作指引 |

### 3.4 与 vmpi 的集成约定

- 管理面代理即透传 `vmpi` CLI（参数原样），boardctl 不重定义管理语义；vmpi 升级不要求 boardctl 跟版。
- 虚拟板 console 暴露：宿主机执行一次 `vmpi port add <consolePort>=<consolePort>`（boardctl doctor 检测到未暴露时给出该指引）。
- LAN 可达性由宿主机侧 vmpi（portproxy 体系）负责，boardctl 不重复造网络层。

## 4. CLI 规格（摘要）

```
boardctl [--board name] [--host H] [--user U] [--port P] [--key K] [--via V] <command>

boardctl doctor [--json]                 # 配置/连通/认证检查
boardctl auth [--password P]             # 首次引导：注入本机公钥
boardctl ssh [command...]                # 交互或执行
boardctl deploy <path> [--svc name] [--args a,b] [--yes]
boardctl console                         # TCP 串口（虚拟板）；真板提示退化
boardctl status [--json] [--svc name]
# 管理面（透传 vmpi，需本地或 --via）
boardctl start|stop|reset|flash|usb ...|port ...
boardctl selftest [--level 0-3|all] [--allow-skip-hardware]
```

破坏性命令确认与 `--yes`、`--json` 输出遵循 REQ-0006。

## 5. 阶段计划与验收标准

> 验收 = selftest 分级用例（REQ-0008），每阶段交付前全绿。

### Q0 骨架
- 范围：Go 工程、cobra CLI、多板配置加载/校验（含单测）、`doctor`（配置与 ssh 连通探测）、selftest L0/L1 骨架。
- 验收：`doctor --json` 对 vmpi 虚拟板输出全部字段；配置错误被明确报告。

### Q1 使用面核心：ssh / status
- 范围：公钥+密码双认证 sshclient；`ssh`（交互/单命令）；`status`（板内视角，--json）。
- 验收：L2 用例——对运行中的 vmpi 虚拟板 `ssh uname -m`=aarch64、status 拉到 uptime/内存；密码认证路径用例通过。

### Q2 deploy 与 auth / console
- 范围：`deploy`（交叉编译/SFTP/systemd/确认机制）；`auth` 引导；`console`（TCP；真板退化提示）。
- 验收：L2——deploy examples（可引 vmpi 的 kvm-lite）→ 服务 active → 端口可访问；auth 后免密登录。

### Q3 agent skill
- 范围：`.agents/skills/boardctl/`（SKILL.md：工作流/守则/诊断；references：命令速查、真板能力矩阵、vmpi 集成说明）。
- 验收：pi agent 加载 skill 后按指令完成一次"部署→验证→诊断"闭环（人工评审核 + 指令清单测试）。

### Q4 管理面 --via 代理
- 范围：`--via ssh://` 透传 vmpi 管理命令（含输出回传与 UAC 类提示的透传）；doctor 检测 via 可达性。
- 验收：L2——经 via 对 vmpi 虚拟板远程 start/stop/status 一轮；无 via 时指引文案正确。

### Q5（条件）MCP server 与 flash
- 范围：REQ-0007 触发后薄封装 MCP（工具 hints）；REQ-0005 的 flash（ssh+dd）危险操作评估。
- 验收：触发时再细化（文档先行）。

## 6. 代码结构（Go）

```
boardctl/
├── README.md
├── docs/DEVELOPMENT.md        # 本文档
├── cmd/boardctl/main.go
├── internal/
│   ├── cli/                   # cobra 命令与全局 flags
│   ├── config/                # 多板配置加载/校验/合并（flag>env>file）
│   ├── sshclient/             # 公钥+密码认证、SFTP、交互 shell
│   ├── deploy/                # 交叉编译、上传、systemd、确认机制
│   ├── viaproxy/              # --via ssh:// 管理面代理（透传 vmpi）
│   └── selfcheck/             # selftest L0–L3
├── .agents/skills/boardctl/   # Q3：agent skill（Agent Skills 规范）
└── tests/
```

依赖约束：Go ≥1.22；cobra、golang.org/x/crypto/ssh、github.com/pkg/sftp；不依赖 vmpi 代码（独立实现，后续如需抽公共 module 另行评估）。

## 7. 流程约定（承接 vmpi）

1. **文档先行**（REQ-0008）：需求确认 → 更新本文档 → 开发。
2. 每阶段完成即更新本文档状态与验收结果。
3. 提交信息格式：`<scope>: <summary>`。
4. 合并前：`go vet`、`go test ./...` 通过；涉板流程的手动验收在提交说明记录。

## 8. 环境要求

| 组件 | 用途 | 阶段 |
|---|---|---|
| Go ≥1.22（Windows/Linux/macOS 任一） | 构建与交叉编译 | Q0 |
| 可达的 vmpi 虚拟板或真板（SSH） | L2 冒烟 | Q1+ |
| 宿主机 OpenSSH（via 代理） | 管理面远程 | Q4 |
