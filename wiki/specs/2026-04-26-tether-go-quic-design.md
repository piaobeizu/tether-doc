# Tether 设计 — Go + QUIC + 三层架构

**Date:** 2026-04-26（PoC-1 验证 2026-04-27）
**Author:** xiaokang.w
**Status:** **Working draft** — 已锁定核心决策；**PoC-1 完成**（12 步全部通过，11 条事实见 Appendix B）；剩余拓扑层决策（§11.A–U）待续讨论

---

## 0. 这份文档是什么

这是 tether v0.1 的完整设计：单用户单机 MVP，技术栈 Go + QUIC + 三层架构（Tauri Mobile App ↔ server ↔ CLI ↔ cc）。心智模型是 daemon 持 PTY + 多 client attach + agent 长存不重启。

PoC-1 已在 2026-04-27 完成（12 步全过、11 条事实见 Appendix B），cc 集成层不确定性已清。剩余的拓扑层决策（§11.A–X）已全部锁定。

**Companion specs**：
- `2026-04-27-phase1-dag-protocol.md` — v0.2 / Phase 1 spec：fenced-block 结构化 UI（dag/form/candidates/media）+ blob proxy + dag skill。本文档 §11.X / D-18 列出 v0.1 为它做的 9 项 forward-compat 预留

> 文档目标：让后续讨论能从**冷启动**接着推。所有锁定的决定写在 §2，所有还没拍板的写在 §11，每个待决项都附了"已知选项 + tradeoff + 当前倾向"。

### 0.1 「Workspace」术语消歧

本 spec 里 **"workspace"** = **tether user workspace**——用户运行 cc 的项目目录（如 `/work/xiyou/`、`/work/wxk.IEBE-1538/`），含 `.tether/` skill 状态、`tether.toml` skill manifest、cc plugin symlink 等。详见 §11.Z / D-19 / Q5-Q6。

不要跟 **polyforge dev workspace**（构建 tether 本身的多 repo 开发环境，根目录 `/root/code/aicoding/tether/`）混淆。后者是 tether 实施期工程基础设施，遵循 polyforge 规范（见 polyforge `references/workspace-layout.md`），不在本设计 spec 范围。

### 0.2 实施期 Polyforge dev workspace 映射

tether v0.1 实施在 polyforge 多 repo workspace 下进行（root：`/root/code/aicoding/tether/`），3 个 repo：

| Repo | role | 内容 |
|---|---|---|
| `tether` | primary | **全部 Go 代码**——CLI 二进制 (`tether`) + server 二进制 (`tether-server`) + wire/envelope schema + E2E crypto + AgentProvider + ClaudeCodeProvider + skill 系统 + daemon goroutine + client goroutine（D-01 两个 binary 都从这一个 Go module 出） |
| `tether-app` | primary | **全部 Tauri / UI 代码**——Tauri 项目（Mobile Android + Desktop macOS/Linux/Windows 多 target）、自写 WT Tauri command（Rust 端 ~500-1000 LOC 包 `web-transport-quinn`）、App UI（Web 端 ~3000 LOC，全 surface fenced block 渲染器 + workspace tree + chat + pair + settings + a11y/i18n） |
| `tether-doc` | doc | **设计 spec / retro / 用户手册**——本文档迁移到 `tether-doc/specs/`；公开发布前的 README / CONTRIBUTING / LICENSE 等 |

**spec 内 `internal/...` Go 路径** 全部相对 `tether` repo 根（即实施期写作 `tether/internal/agent/claudecode/spawn.go` 等）。
**spec 内 Tauri / UI 引用** 全部位于 `tether-app` repo。
**用户侧路径**（`~/.tether/skills/<skill>/` 全局池、`<workspace>/.tether/<skill>/` skill 状态、`<workspace>/.claude/plugins/<skill>` cc plugin 符号链接）跟 polyforge dev workspace **无关**——是 tether 运行时给用户使用的，跟实施工程目录两套独立概念。

---

## 1. 三个核心架构选择

### 1.1 传输：QUIC

启用 QUIC 的三个核心能力：
- **Connection migration**：手机 WiFi↔4G 漂移、笔记本休眠唤醒，不需要重连
- **多 stream**：control / 输入 / 输出 / replay 各走独立 stream，慢 stream 不阻塞快 stream
- **Datagrams**：presence / VAD / 心跳走不可靠 datagram，省去 TCP 排队

库选型：`github.com/quic-go/quic-go`（事实标准，2026-01 最新版本，覆盖 RFC 9000/9001/9002 + HTTP/3 + QPACK + Datagrams RFC 9221）+ `github.com/quic-go/webtransport-go`（WT server）。浏览器侧用 WebTransport（QUIC 的浏览器封装），Chrome/Edge/Firefox/Safari 26.4+ 都支持。

**不做 WS fallback**（详见 §11.A）。CLI 和 Mobile App 都走 WebTransport-over-HTTP/3，server 单一 listener，wire schema 对称（详见 §11.T）。**App 端（Android + Desktop + 未来 iOS）四平台统一**走 tether 自写 Tauri command 包 `web-transport-quinn`（kixelated 维护活跃 lib；review 2026-04-28 P1 #5 = ii-rebuilt；详见 §11.V）。

### 1.2 语言：Go

- 单二进制静态链接（5–15MB），无 npm + node-pty native 编译之苦
- CLI 和 server 同语言，wire schema 可直接 import
- goroutine + channel 让 main+daemon+client+watchdog 这套 supervision 模型实现非常自然
- Mobile App 的 JS/TS 类型可以从 Go 通过 protobuf/json schema 生成

Node.js / Rust 已评估后驳回（详见 §12.B / §12.C）。

### 1.3 拓扑：三层

```
Mobile App ──→ server ──→ CLI(daemon+client) ──→ cc
                                         └─→ cc
```

收益：
- 用户机器可以躲在 NAT 后（CLI 走 outbound 连 server）
- server 可暂存离线消息
- E2E 模型对称
- 给 v0.2 多用户/多机预留空间

代价：server 公开可达。

---

## 2. 已锁定的决策（不再重开）

| # | 决策 | 锁定值 |
|---|---|---|
| D-01 | CLI 与 server 语言 | **Go** 。CLI 是单二进制 (`tether`) 多子命令；server 是另一个二进制 (`tether-server`) |
| D-02 | 传输协议 | **QUIC**：CLI 和 Mobile App 都走 **WebTransport-over-HTTP/3**，server 单一 listener，wire schema 对称。**不做 WS fallback**——PoC-2 是硬 gate，失败时降级为 raw QUIC HTTP/3 长连接（仍纯 UDP/QUIC，不引入 TCP/WS）。**App 端 transport（review 2026-04-28 P1 #5 = ii-rebuilt）**：tether 自写薄 Tauri command 包 `web-transport-quinn`（kixelated 维护，活跃 lib）+ 简化 JS shim（不抄 W3C `WebTransport` IDL，仅暴露 4-6 个 minimal command），跑遍 Android + macOS + Linux + Windows 4 个 target；iOS v0.1.x 复用同一份代码。**不依赖任何 abandoned plugin**——`tauri-plugin-web-transport`（kixelated 那个 4 commit hobby 项目）已确认废弃，不采用。详见 §11.V |
| D-03 | 整体拓扑 | **三层** Mobile App ↔ server ↔ CLI |
| D-04 | v0.1 范围 | 单用户单机：server 只服务一个用户、CLI 只跑在一台机器上。userId / machineId 字段保留以便 v0.2 |
| D-05 | Agent 交互方式（v0.1 ClaudeCodeProvider 实现）| **PTY (mode ①) + Hooks (mode ③) + JSONL watcher**。**禁用** SDK 与 stream-json 模式。这是 cc provider 的具体实现选择；其他 provider（codex/opencode/...）按各自合适的方式实现 AgentProvider 接口。详见 D-17 |
| D-06 | PTY 持有（cc provider） | daemon goroutine 自持，用 `github.com/creack/pty`。**不引入** abduco / dtach / tmux 等容器。其他 provider 不一定走 PTY（看 agent CLI 的设计） |
| D-07 | Daemon 死亡语义 | 接受"daemon 死 → agent 死"，靠 `tether resume <id>` + 该 provider 的 replay 机制（cc 用 JSONL replay）兜底；停机感知 5–10s |
| D-08 | 切换心智模型 | **agent 长存不重启**，切换 = 多 client 共 attach。明确**与 happy 的"杀进程切 SDK"模型不同**。所有 AgentProvider 实现都必须满足这个性质 |
| D-09 | Lock 强制性 | 真机制：所有 agent 输入必须经过 daemon 内存中的 input gate；不存在 advisory 路径 |
| D-10 | 终端入口 | `tether attach <sid>` 自定义 raw-mode TTY 客户端。**不允许**用 native `cc`（或其他 agent CLI）直接进入活会话 |
| D-11 | CLI 进程结构 | `main goroutine = watchdog`；下辖 `daemon goroutine`（agent provider/PTY/lock）+ `client goroutine`（QUIC 连 server）。两个子 goroutine panic 后 main 自动重启它们；长期资源（PTY fd / listener / 共享状态）由 main 持有 |
| D-12 | E2E 加密 | **完整 E2E（C1 精简版）**：X25519 配对 + XChaCha20-Poly1305 envelope + HKDF-SHA256 派生；device-pair wrap_key + 每 cc-session 持久化 session_key（**server-compromise confidentiality**，非完整 forward secrecy——见 §11.C）；server 看不到 plaintext。详见 §11.C |
| D-13 | App 壳子（Mobile + Desktop） | **v0.1 ship 两个 target：Android（Tauri 2 Mobile）+ Desktop（Tauri 2 Desktop，覆盖 macOS / Linux / Windows）**。同一份 Tauri 项目 + WebView UI 代码，build matrix 出多 target；fenced block 渲染器 100% 共享，仅 layout container 不同（手机单栏 / 桌面三栏）。**Transport 四平台统一 homebrew polyfill**（review 2026-04-28 P1 #5 = ii-rebuilt）：所有 target 通过 **tether 自写 Tauri command** 包 `web-transport-quinn`（kixelated 维护，活跃 lib）做 WebTransport，**不依赖 WebView 原生 WT 也不依赖 abandoned `tauri-plugin-web-transport`**——transport implementation 100% 对称，调试 / 测试矩阵减半，Chromium WebView 升级周期解耦。Android = FCM push + Android Keystore；Desktop = 各平台 keychain（macOS Keychain / libsecret / Windows Credential Manager）。**Desktop 永远是 daemon 的远程客户端**（与 §11.I I1 部署一致：daemon + server 同机部署在 GCP VM / 工作站；详见 §11.Y.5 D-8 决议）——v0.1 无 desktop ↔ daemon 同机 UDS 直连分支。**iOS 推迟到 v0.1.x**（同一份 tether-自写 Tauri command 复用；增量在 APNs + Apple Developer 流程 + iOS 真机 smoke test）。不出 PWA / 浏览器版。**前置依赖 PoC-2.6 Android smoke test**（验证 `web-transport-quinn` × `aarch64-linux-android` 编译 + 真机 / emulator 基本 WT dial；详见 §10）。详见 §11.G / §11.V / §11.Y |
| D-14 | 通信模型 / Wire 协议 | **Envelope-based**（明文 routing 元数据 + E2E ciphertext payload，JSON 编码 v0.1）。WT **5 通道**：control / events / agent-bytes / catch-up / datagram，对应 HTTP/3 PRIORITY 区分高/低。Mobile App **只渲染原生 chat UI**（数据来自 cc 自写的 JSONL，不解析 PTY 字节）；**Stream 3 agent-bytes v0.1 不在 WT 上消费**（仅本地 `tether attach` 走 daemon Unix socket）。`output.agent-bytes` chunk 化策略（16KB/100ms）+ Backpressure trim 策略 spec 完整保留以备 v0.1.x 远程 attach。详见 §3.3 |
| D-15 | Lock 状态机 | **单层 lock（writer）+ 60s auto release + force takeover (`Ctrl+\ Ctrl+T` / Mobile UI 二次确认)**。**View-only attach v0.1 不做**——多 client 已收广播 envelope 默认就能"看"。Permission_mode 切换走 RPC（`session-set-mode`），权限基于 lock holder 身份。详见 §11.D |
| D-16 | Auth Token 复杂度 | **J2 简化版**：access token (~1h) + refresh token (~30day)。**无 token-family / grace-window / divergence detection**（happy 多用户场景才需要）。盗用场景靠 `tether revoke <deviceId>` 重新配对。配对协议本身用 D-12 / C1 的 X25519 ECDH + QR 直扫。详见 §11.J |
| **D-17** | **AgentProvider internal seam（v0.1 不公开多 provider 承诺）** | daemon 内部用 `AgentProvider` Go interface 隔离 cc-specific 逻辑，**v0.1 仅 `ClaudeCodeProvider` 一个实现**。**不暴露公开多 provider 能力**——无 `tether agents` CLI、无 `--agent` flag、无"加新 provider 指引"承诺。接口形态在第二个 provider 实际实作时再稳定。Wire envelope 仍用 `output.agent-event` + `providerType` 字段，但 v0.1 该字段恒等 `"claude-code"`。详见 §11.W |
| **D-18** | **v0.2 Phase 1 forward-compat 预留**（路径占位 + 关键 seam，**不承诺"DAG ready"**）| v0.2 / Phase 1（参考 `2026-04-27-phase1-dag-protocol.md`）将引入 fenced-block 结构化 UI + button-to-slash + blob proxy。v0.1 预留 9 项口子（workspaceRoot / `/hooks/` 前缀 / `/blob/*` 路由名 / blob E2E contract / App 渲染 pipeline / `sendUserMessage` API / `.tether/` 自动建 / `tether-blob-register` stub / `catchup.state-snapshot` kind）共 ~135 LOC——**这些是路径/字段占位 + App 关键架构 seam，避免破坏性升级**；v0.2 真正难点（streaming FSM、4 类 component renderer、按钮安全、blob auth/cache、状态重放）仍是 v0.2 实作期工作量。详见 §11.X。**注**：D-19 锁定后，#5 项（App parseContent → renderBlock pipeline）从"预留 seam"升级为"v0.1 1st-class 实现"——见 §11.Y |
| **D-19** | **App surface model（桌面三栏 + 移动端 chat-first 压缩 + 共享 fenced block 渲染器）** | tether 定位 = **AI-generalized VS Code**：桌面三栏（左 workspace 树 / 中 skill 产物 + actions / 右 AI chat）作主形态；移动端 = 同一份 fenced block 渲染器在 commute-mode 单栏 layout 下的压缩（chat 流为主 + skill 产物 inline card + 抽屉藏 workspace 树）。**fenced block 协议（dag/form/candidates/media）从 v0.2 提前到 v0.1 1st-class**——desktop 中栏 + 桌面右栏 chat + 手机 chat 流复用同一渲染器。skill 作者只 emit fenced blocks，不关心两端 layout。详见 §11.Y |
| **D-21** | **Desktop / Mobile 统一 transport（D-8）** | desktop 永远是 daemon 的**远程**客户端（与 §11.I I1 部署一致：daemon + server 同机在 GCP VM / 工作站，desktop / mobile 都通过 WT-over-HTTP/3 远程接入）。**v0.1 无 UDS 直连分支** —— desktop 跟 mobile envelope wire 100% 对称，单一 `@/transport` implementation。`tether attach` 仍走 daemon Unix socket 但只能在 daemon 同机 shell 跑（SSH 进 VM）。详见 §11.Y.5 |
| **D-20** | **Skill 模型 = cc plugin + 薄薄一层 `tether.toml` overlay**<br/>（含 B-4 状态归属 + B-5 active skill 切换 + B-6 缺失 fallback + P1 #12 runtime 拥有权）| tether skill **不发明新格式**——直接复用 cc plugin 标准结构（`skills/<name>/SKILL.md` + `hooks/hooks.json` + `commands/` + `scripts/` + `agents/`）。daemon 不直接 spawn / sandbox skill 进程：所有 skill 执行 **delegate 给 cc 自己的 plugin loader + hook 系统**。tether 仅加一个**可选**补充 manifest `tether.toml` 声明 tether-specific 元信息（emitted fenced blocks / mobile UI hints / state dir / agent target）。**Hook event 名 v0.1 不做命名空间化**（review 2026-04-28 P0 #4 = c，YAGNI）——skill 的 `hooks/hooks.json` 直接用 cc 标准 event 名（`PreToolUse` 等），polyforge 现有 16 个 skill 不改一行。Adapter 模块 `internal/skill/adapter/{cc,codex,opencode,aider,cursor}.go` 从 day 1 命名存在——v0.1 仅 cc 真实现（pure symlink，~70 LOC）。**B-4 状态归属**：skill 自管 `<workspace>/.tether/<skill>/`，daemon 不读；daemon 只缓存最近 fenced block 渲染快照（每 (session, skill) 20 个，全局 1000/10MB LRU）。**B-5 active skill**：所有 enabled skill 同时 active，无"模式切换"；fenced block 通过 **fence tag 后缀语法** ` ```<type>:<skill> ` 携带 skill 身份（详见 §11.AA.1）；daemon 只 grep fence 行不 parse body，保留 D-20 daemon-不感知-skill 承诺；按时间序混排进 chat。**B-6 skill 缺失**：默认 fail with helpful error（不 implicit clone）；`--auto-install` / `--allow-missing` 显式 opt-in。**P1 #12 Runtime 拥有权**：tether **拥有** cc runtime——production 假设独立容器零 user-level skill；v0.1 dev 模式过渡，cc 默认 merge user-level + workspace-level，用户自管。详见 §11.Z |

---

## 3. 架构总览

### 3.1 拓扑

```
┌──────────────────────────────────────────────────────────────────────┐
│ User's machine（笔记本 / 工作站 / GCP VM）                           │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐       │
│  │ tether 进程（Go 单二进制）                                 │       │
│  │                                                            │       │
│  │  main goroutine = watchdog                                 │       │
│  │  ├─ 持有：PTY master fd、Unix listener、SharedState、ctx   │       │
│  │  └─ 监督：daemon、client                                    │       │
│  │                                                            │       │
│  │  ┌──────────────────────┐    ┌─────────────────────────┐  │       │
│  │  │ daemon goroutine      │    │ client goroutine        │  │       │
│  │  │                       │    │                         │  │       │
│  │  │ • AgentProvider       │    │ • QUIC outbound 连 server│  │       │
│  │  │   (v0.1: ClaudeCode)  │ ←─►│ • 收下行 envelope       │  │       │
│  │  │ • Lock state machine  │ ch │ • 推上行 envelope       │  │       │
│  │  │ • input gate          │ ans│ • 自动重连 + 心跳       │  │       │
│  │  │ • Unix socket（local  │ els│ • QUIC connection       │  │       │
│  │  │   tether attach 入口）│    │   migration auto-handles│  │       │
│  │  │                       │    │   network change        │  │       │
│  │  └────────────┬──────────┘    └─────────────────────────┘  │       │
│  └───────────────┼──────────────────────────────────────────────┘     │
│                  │                                                    │
│                  ▼  AgentProvider internal seam（D-17）              │
│            ┌─────────────────────┐                                    │
│            │ ClaudeCodeProvider  │ ← v0.1 唯一实现，不公开            │
│            │ • PTY (creack/pty)  │                                    │
│            │ • Hook HTTP server  │                                    │
│            │ • JSONL watcher     │                                    │
│            └──────────┬──────────┘                                    │
│                       ▼                                                │
│                   ┌──────┐                                            │
│                   │  cc  │                                            │
│                   └──────┘                                            │
│                                                                       │
│  ┌──────────────────────┐                                             │
│  │ tether attach <sid>  │ ← 同机器另一个 tether 子进程                 │
│  │ raw-mode TTY 客户端   │   连 daemon goroutine 的 Unix socket        │
│  └──────────────────────┘                                             │
└──────────────────────────────────────────────────────────────────────┘
                            │
                            │ QUIC outbound
                            ▼
┌──────────────────────────────────────────────────────────────────────┐
│ tether-server（Go 二进制；可与 CLI 同机部署，也可独立部署）           │
│                                                                      │
│  • WebTransport-over-HTTP/3 listener（CLI + Mobile 共用单 listener）  │
│  • HTTP/3 GET `/blob/<sha256>` 路由保留（D-18，v0.2 Phase 1 启用）    │
│  • Auth / 配对（包含跨 CLI/Mobile App 的 device pubkey 关联）         │
│  • Session/Machine 注册（v0.1 各只有 1 个，但 schema 已多元化）       │
│  • Envelope 路由：Mobile→CLI 和 CLI→Mobile 之间的转发                 │
│  • 离线消息暂存（CLI 断开时 Mobile 发的消息存起来，CLI 上线 replay）  │
│  • **不参与 push**：CLI 自己直推 APNs/FCM（D-12 / E2 决议）           │
│  • 不解密内容：E2E C1，server 是哑中继（D-12）                        │
└──────────────────────────────┬───────────────────────────────────────┘
                               │ WebTransport-over-HTTP/3
                               ▼
┌──────────────────────────────────────────────────────────────────────┐
│ Phone（Tauri Mobile App：iOS / Android 原生壳 + WebView 内 React UI） │
└──────────────────────────────────────────────────────────────────────┘
```

### 3.2 组件清单

| 组件 | 处理 |
|---|---|
| 用户机器上的 daemon | 三层中的"用户机器"层；持 cc + PTY + lock + JSONL watcher |
| `tether attach` 客户端 | 自定义 raw-mode TTY，连 daemon 的 Unix socket |
| **独立 server** | WebTransport-over-HTTP/3 listener（CLI + Mobile 共用），envelope 路由 + 离线暂存。**不参与 push** |
| Mobile App | **Tauri 2 Mobile**（iOS + Android 原生壳）+ React + Vite（WebView 内）。详见 §11.G / §11.V |
| PTY 库 | `github.com/creack/pty`（不是 node-pty）|
| App transport（Mobile + Desktop） | **纯 WebTransport-over-HTTP/3**；四平台（Android + macOS / Linux / Windows）统一走 tether 自写 Tauri command 包 `web-transport-quinn`（review 2026-04-28 P1 #5 = ii-rebuilt；详见 §11.V）|
| Push | **CLI 直推 APNs / FCM**（E2 决议，§11.E）；Pusher 多态抽象 |
| Daemon 多 client fan-out | daemon 自管 ring buffer，goroutine + channel 实现 |

### 3.3 通信模型（Wire 协议）

D-14 锁定的 wire 层全貌。E2E 加密（D-12）+ Tauri Mobile（D-13）+ WebTransport（D-02）共同决定的产物。

#### 3.3.1 Envelope schema

```jsonc
{
  // ─── 路由元数据（明文，server 看得到，路由 + 限流用）───
  "id":           "cuid2-...",      // 全局唯一 envelope ID（dedup / ack）
  "sessionId":   "uuid-...",        // cc session ID（路由 + AD）
  "fromDeviceId": "device-cli-1",   // 发送方设备
  "toDeviceId":  "device-app-2" | "*",  // "*" = 同 user 所有 device 广播（不含发送方）
  "ts":          1714000000000,     // 发送时间戳（防 replay 窗口）

  // ─── 操作语义（明文，server 路由 + 优先级判断要看 kind）───
  "kind":         "input.user-message",    // 见 §3.3.2 操作分类
  "reqId":        "cuid2-...",      // 可选：RPC 模式下这条 envelope 的请求 ID
  "inResponseTo": "cuid2-...",      // 可选：RPC 响应 envelope 引用 reqId
  "seq":          42,               // 可选：流式消息的单调序号（per-(sessionId, kind)）

  // ─── 加密层（D-12 / C1）───
  "keyVersion":   1,                // v0.1 恒定 = 1；密码学协议升级才 bump
  "nonce":        "base64...",      // 24B XChaCha20 nonce
  "ciphertext":   "base64..."       // XChaCha20-Poly1305(session_key, nonce, payload, AD)
                                    // AD = sessionId || fromDeviceId || toDeviceId || keyVersion || kind
}
```

**关键设计点**：
- `kind` 在明文外层是**故意的**——server 路由 + WT stream priority 决策需要看 kind。但 kind 不暴露 plaintext 内容
- AD 把 `kind` 包进去，防"server 改 kind 重路由"——保留 E2E 完整性
- Encoding：v0.1 用 **JSON**（debug 友好、Go/TS 都顺）；v0.2 看带宽再考虑 CBOR / protobuf（5-10x 体积减少）
- `seq` per-(sessionId, kind) 单调——Mobile App 端用它去重和检测 gap

**Replay / dedup 防御**（P1-7 review 加强）：
- 接收端（Mobile App / CLI / Server）必须维护 **replay window**：
  - **时间窗**：`|now - envelope.ts| > 5 minutes` → 拒绝并告警（防 server 持有的 ciphertext 被恶意延迟 replay）
  - **ID 去重缓存**：最近 N=10000 个 envelope `id` 的 LRU set，命中即拒绝（防 server 重发已交付 envelope）
  - **per-stream seq monotonic**：同 stream 内 `seq` 必须严格递增——倒序到达即为攻击或 bug
- Server 端**仅做 routing**，不主动 replay——但 server 暴露的"重发 catch-up envelope" 能力受约束：catch-up envelope 必须是之前确实 emit 过的（按 `id` + `seq` 双重检查）
- **Threat model 显式化**：server 入侵 = traffic analysis（kind 字段、活跃时间、deviceId 关系图）+ replay/delay risk（受 5min 时间窗 + ID dedup 限制）+ ciphertext 持久化（不解密但可永久存档供未来 cryptanalysis）。**不是**简单的 "DoS only"

#### 3.3.2 Operation taxonomy

按"流量方向 + 用途"分组，6 大类：

**Session 生命周期**（control stream）
| kind | 方向 | 语义 |
|---|---|---|
| `session.create` | App → CLI | 起新 cc session |
| `session.resume` | App → CLI | resume `needs-resume` 的 session |
| `session.end` | 任一 → 对端 | 关 cc session（rotate session_key） |
| `session.state` | CLI → App broadcast | 状态变化（lock holder / mode / status） |

**输入**（control stream）
| kind | 方向 | 语义 |
|---|---|---|
| `input.user-message` | App → CLI | 用户消息文本（PTY 写） |
| `input.cancel-turn` | App → CLI | §5.3 cancel-turn 序列 |
| `input.pre-tool-approval` | App → CLI | PreToolUse hook 的 allow/deny 响应 |

**输出 - 事件**（events stream，**永不丢**）
| kind | 方向 | 语义 |
|---|---|---|
| `output.agent-event` | CLI → App | provider-normalized 结构化事件。Plaintext payload 含 `providerType` + `event` 字段，App 按 providerType 分发渲染器（cc 的 user/assistant/tool_use 等只是 ClaudeCodeProvider 的一种 schema）|
| `output.hook-event` | CLI → App | provider 拦截点触发（如 cc 的 PreToolUse 待审批）。`providerType` 字段决定 hook 语义 |

**输出 - 字节流**（agent-bytes stream，**可丢**）
| kind | 方向 | 语义 |
|---|---|---|
| `output.agent-bytes` | CLI → App | provider 字节流 chunk（cc 是 PTY 字节流；其他 provider 可能是 stdout 字节）。16KB / 100ms 双阈值切片。**v0.1 不上 WT** |

**控制**（control stream）
| kind | 方向 | 语义 |
|---|---|---|
| `control.lock-request` | App / 远程 attach → CLI | 申请 lock |
| `control.lock-takeover` | 同上 | 强制接管 |
| `control.set-mode` | lock holder → CLI | 切 permission_mode |
| `control.catchup-needed` | server → 慢 client | "已 trim N 条 agent-bytes，请 catch-up" |
| `control.error` | 任一 → 对端 | 协议错误 |

**Catch-up**（catch-up stream，server→client 单向）
| kind | 方向 | 语义 |
|---|---|---|
| `catchup.event` | server → 重连/慢 client | replay 离线/被 trim 期间错过的 envelope（events 永不丢；agent-bytes 跳过被 trim 的，靠 agent redraw 重画）|
| `catchup.state-snapshot` | server → client | **v0.2 Phase 1 预留**（D-18）：workspace state 快照（如 dag skill 的 `.tether/dags/*.json`），让 App 重连时跳过中间状态直接拿最终态。v0.1 不实现 |
| `catchup.done` | server → client | catch-up 完成，回到正常流 |

**健康**（datagram，unreliable）
| kind | 方向 | 语义 |
|---|---|---|
| `health.heartbeat` | 双向 | 每 15s 一次 |
| `health.presence` | CLI → App | "我在线"（含 lock 状态） |

#### 3.3.3 Stream / Datagram 分配（5 通道）

QUIC stream 内严格顺序，所以**优先级和"是否可丢"必须在 stream 层拆分**——不能在同一个 stream 里"插队"。

| Stream | 用途 | HTTP/3 priority | 丢弃策略 | v0.1 是否实际订阅 |
|---|---|---|---|---|
| **Stream 1** (control) | `session.*` / `control.*` / `input.*` | **HIGH** | 永不丢 | ✅ Mobile App + 未来远程 attach |
| **Stream 2** (events) | `output.agent-event` / `output.hook-event` | **HIGH** | 永不丢 | ✅ Mobile App + 未来远程 attach |
| **Stream 3** (agent-bytes) | `output.agent-bytes` | **LOW** | backpressure 下 trim | ⏸ **v0.1 不上 WT**（仅 daemon 本地 Unix socket→`tether attach` 用）|
| **Stream 4** (catch-up) | `catchup.*`（server→client 单向） | MEDIUM | 不丢（replay 走慢点没关系） | ✅ Mobile App |
| **Datagram** | `health.*` | — | 天然 unreliable | ✅ 双向 |

**Stream 3 agent-bytes 在 v0.1 的状态**：

- **Mobile App 不订阅** —— App 渲染原生 chat UI（React 组件），数据来自 Stream 2 的 `output.agent-event`，**不需要解析 PTY 字节**（不需要 xterm.js）
- **`tether attach` 走本地 Unix socket** —— daemon 通过 ring buffer fan-out 给本地 attach 客户端；不上 WT
- v0.1 阶段 daemon 不必往 stream 3 emit agent-bytes（节省 Mobile 流量）
- 协议位置**保留**——未来 v0.1.x 加 `tether attach --remote` 或 Mobile 加 raw TUI 视图时启用

**为什么必须拆 stream 2 (events) 和 stream 3 (agent-bytes)**（即使 v0.1 不用 stream 3）：
- 协议层面前瞻，避免 v0.1.x 引入 agent-bytes 时改 wire schema
- cc 大段输出（如 `cat` 大文件 / 生成大段代码）一次产生 ~100KB 字节流
- 4G 上行 ~1Mbps → 100KB 串行 ~800ms
- 同 stream 时 hook 审批排在后面要等 800ms+——通勤场景不可接受
- 拆 stream 后 events stream 不被 agent-bytes 阻塞

#### 3.3.4 Chunk + ack + trim 策略

**Chunker（CLI 端，daemon 内）**：

```go
const (
    chunkSizeMax = 16 * 1024            // 满 16KB 立即发
    chunkTimeMax = 100 * time.Millisecond // 不满也每 100ms 强制 flush
)
```

- 稳态低速（1-10 KB/s）：每 100ms 一个小 chunk，metadata overhead 高但绝对带宽小
- 峰值（100 KB/s）：每填满 16KB 立即发，overhead 1%
- 大段 burst：填满即发，不等 100ms

**Ack 机制**：
- `output.agent-bytes` 不需要单独 ack（trim 才需要触发 catch-up）
- `output.agent-event` / `output.hook-event` 走 stream 2，QUIC stream-level 顺序送达即足够
- RPC 类（`input.pre-tool-approval` 之类）通过 `inResponseTo` 字段 ack

**Trim 策略**（5 层 backpressure）：

| Layer | 何处 | 慢消费者怎么办 |
|---|---|---|
| L1 | PTY → daemon ring buf | **不能阻塞 PTY reader**。daemon 异步 drain，**ring buf 4MB**（升 1MB → 4MB）；overflow 行为：**丢最老 agent-bytes 数据**（保 events 已经在 stream 2，ring buf 仅承载 PTY raw bytes）+ 上报 `degraded` 状态（emit `control.error{kind:"ringbuf-overflow"}`）。**绝不阻塞 PTY reader goroutine**（否则 cc stdin 卡 → cc 自己 hang）。**PoC：cc 5MB / 50MB 单次输出 stress test 必跑** |
| L2 | daemon → 多 client fanout | 慢 client channel 满 → **trim 老 agent-bytes**（保 events）+ 该 client 发 `control.catchup-needed` |
| L3 | daemon → server WT stream | QUIC stream flow control 自然反压，不主动处理 |
| L4 | server → App WT stream | server 暂存满（10MB/session）→ 同 L2 trim 策略 |
| L5 | App WebView 渲染 | 不做 backpressure；envelope 立即 ack 让上游释放；React virtualization |

**Trim invariant**（保不变量）：
- `output.agent-event` / `output.hook-event` / `control.*` / `session.*` / `input.*`：**永不丢**
- `output.agent-bytes`：可丢（cc 自己会 redraw 一屏 TUI）

**Trim → catch-up 流程**：

```
daemon L2 / server L4 检测到 channel/queue 接近满 →
  evictOldest(kind == "output.agent-bytes")  // 删掉老的 agent-bytes envelope
  emit `control.catchup-needed` { lostFromSeq: <最后被丢的 seq> }

接收方收到 control.catchup-needed →
  emit `session.resume` 触发 stream 4 catch-up
  → server 重发 lostFromSeq 之后的 events（agent-bytes 不重发，cc redraw 即可）
```

#### 3.3.5 三个核心 Flow Walkthrough

**Flow 1：Hook 审批（PreToolUse）**

```
T0  cc 想跑 `git push`
T1  cc → PreToolUse hook → daemon HTTP 端 (POST)
    daemon HTTP handler 阻塞等响应
T2  daemon emit `output.hook-event` envelope (Stream 2 events)
    daemon 同时通过 §11.E E2 推 push (CLI 直推 APNs/FCM)
T3  App 收到 envelope + push notification
    用户点 push → App 打开 → 显示对话框
T4  用户点 Allow
    App emit `input.pre-tool-approval` envelope (Stream 1 control)
    reqId = T2 envelope 的 id；inResponseTo = T2 envelope 的 id
T5  daemon 收到 → HTTP 响应 cc with `permissionDecision: "allow"`
T6  cc 跑 git push
```

**Flow 2：Session Resume（daemon 重启场景）**

```
T0  daemon 死了 → cc 也死 → meta.json 标 session 为 needs-resume
T1  App 重连 server（QUIC 自动重连）
    server 推 `session.state { status: "needs-resume" }` (Stream 1 control)
T2  App 显示 banner "点击恢复"
T3  用户点 → App emit `session.resume` (Stream 1 control)
T4  CLI（已重启的 daemon）收到 → 从 keys.bin 恢复 session_key（B4 / D-14）
    spawn cc with --session-id
    daemon JSONL watcher 起来 → 增量发 `output.agent-event` (Stream 2)
T5  cc TUI 渲染 → daemon emit `output.agent-bytes` chunks (Stream 3)
T6  App TUI 区域同步、消息列表同步
```

**Flow 3：App 离线 5 分钟后 catch-up**

```
T0  App 切走 → WT 连接断
T1  CLI 持续 emit envelope → server 暂存（B4，TTL 1h，10MB/session）
    server 端如果暂存接近上限 → trim 老 agent-bytes、保 events
T2  App 重连 → 建 WT session
T3  App emit `session.resume`，带 lastSeenEventUuid
T4  server 在 stream 4 emit `catchup.event` × N（重发 lastSeen 之后的 events 全集，agent-bytes 选择性发）
T5  server emit `catchup.done`
T6  App 切回正常监听 stream 2 / 3
    cc redraw 一屏 TUI → stream 3 重发当前完整 TUI
    server 删除已 ack 的 envelope
```

#### 3.3.6 LOC 估算

| 模块 | LOC |
|---|---|
| Envelope schema 定义（Go + TS schema 共享） | ~100 |
| Stream 生命周期（5 通道，建立 + 关闭 + 错误） | ~250 |
| Chunker（PTY → agent-bytes envelope） | ~50 |
| Reassembler（App 端 agent-bytes envelope → TUI 字节流） | ~40 |
| Trim 逻辑（daemon L2 + server L4） | ~150 |
| Catch-up 协议（client + server 两侧） | ~120 |
| RPC 层（reqId / inResponseTo 匹配 + 超时） | ~100 |
| **合计** | **~810 LOC** |

---

## 4. CLI 进程模型详细

### 4.1 watchdog pattern 的承诺与限制

**能挡住的崩溃场景**（约 90% 实际故障频率）：
- daemon 子系统代码中的 panic（nil deref、index OOB、未初始化 map） → recover + restart goroutine
- daemon 子系统死循环 / 死锁 → heartbeat 超时检测 + 强制 cancel ctx + restart
- client 子系统的 QUIC / 加密协议解析 panic → 同上

**挡不住的**（约 10% 实际频率）：
- OOM（kernel SIGKILL 整 process）
- `kill -9 <pid>`
- Go runtime fatal（stack overflow、并发 map 写、runtime bug 等不可 recover 的）
- OS panic / VM 重启 / 断电

**剩下这 10% 通过 D-07 的 resume 兜底覆盖**。两层防线叠加，目标用户感知崩溃频次 ≤ 每周 1 次。

### 4.2 资源所有权规则（重要）

```
main goroutine 持有（process-lifetime）：
  • PTY master fd（cc 不会因为 daemon goroutine panic 而 SIGHUP）
  • Unix socket listener（daemon goroutine 重启不丢已连接客户端的可能性，待评估）
  • envelope 出/入 channel（fromDaemon、toDaemon），子 goroutine 通过引用使用
  • SharedState 结构体（Lock、Session map、Device map），由 sync.Mutex / sync.RWMutex 保护
  • Context（cancel 时通知所有子系统优雅退出）

daemon goroutine 持有（goroutine-lifetime，重启时丢弃重建）：
  • JSONL UUID 索引（重启时从磁盘重建）
  • PTY ring buffer（重启时从空开始；cc 会 redraw）
  • Hook HTTP server（重启会短暂中断 hook 调用，cc 自带重试）
  • PTY 读循环 goroutine 引用

client goroutine 持有（goroutine-lifetime，重启时丢弃重建）：
  • QUIC Connection 对象（panic 后连接状态可能损坏，必须丢弃重连）
  • 重连状态机
  • 心跳 timer

⚠️ 关键：QUIC Connection 不在 main 里持有，因为 panic 后 connection 的加密 / stream 状态可能进入半失败态，复用反而引入隐患；丢掉重连（QUIC 0-RTT 让重连成本低）更稳。
```

### 4.3 强制纪律（CI 必须 lint）

子系统 goroutine 中**禁止**：
- `log.Fatal*` / `log.Panic*`
- `os.Exit`（仅 main 可调）
- 启动新 goroutine 时不包统一的 `goSafe()` panic 处理 helper
- 直接 `panic(...)` 在没有 defer recover 的路径上

实施方式：custom analyzer（基于 `golang.org/x/tools/go/analysis`），在 CI 跑。

### 4.4 骨架代码（仅供讨论参考）

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    // 进程级资源
    ptmx, claudeCmd := startCC(ctx, sessionConfig)
    ctrlListener   := startUnixSocketListener(localControlSocketPath) // 给 tether attach 用
    sharedState    := newSharedState()

    // envelope 通道
    fromDaemon := make(chan Envelope, 256)
    toDaemon   := make(chan Envelope, 256)

    superviseLoop(ctx, "daemon", func(ctx context.Context) error {
        return runDaemon(ctx, ptmx, ctrlListener, sharedState, fromDaemon, toDaemon)
    })

    superviseLoop(ctx, "client", func(ctx context.Context) error {
        return runClient(ctx, sharedState, fromDaemon, toDaemon)
    })

    // 等 SIGTERM/SIGINT
    waitForShutdown(cancel)

    // graceful：通知 cc 退出 + 等子 goroutine 收尾
    gracefulShutdown(ptmx, claudeCmd)
}

func superviseLoop(ctx context.Context, name string, fn func(context.Context) error) {
    go func() {
        backoff := backoffStrategy()
        for {
            err := safeRun(fn, ctx)
            if ctx.Err() != nil {
                return
            }
            log.Errorf("%s subsystem: %v, restarting in %v", name, err, backoff.Next())
            time.Sleep(backoff.Next())
        }
    }()
}

func safeRun(fn func(context.Context) error, ctx context.Context) (err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("panic: %v\n%s", r, debug.Stack())
        }
    }()
    return fn(ctx)
}
```

---

## 5. Agent 交互模型

> **架构层级**：daemon 内部用 `AgentProvider` Go interface 隔离 cc-specific 逻辑（D-17 internal seam）。**v0.1 只有 `ClaudeCodeProvider` 一个实现**，不公开多 provider 能力（无 `--agent` flag、无 `tether agents` CLI）。§5.0 介绍 internal seam；§5.1 - §5.6 描述 ClaudeCodeProvider 实际行为。

### 5.0 AgentProvider internal seam（Go interface）

daemon 内部 ` internal/agent` 包定义 Go interface 隔离 cc-specific 逻辑：

```go
type AgentProvider interface {
    ProviderType() string                                          // v0.1 恒等 "claude-code"
    Spawn(ctx, opts SpawnOpts) (*AgentSession, error)
    Resume(ctx, sessionID string) (*AgentSession, error)
    Kill(session *AgentSession) error
    SendUserMessage(session *AgentSession, text string) error
    CancelTurn(session *AgentSession) error
    Events(session *AgentSession) <-chan AgentEvent                // normalized 流
    SupportsHooks() bool
    HookCapabilities() []HookCap                                   // 声明拦截点 + payload schema
    RespondHook(hookID string, decision HookDecision) error
    SupportedModes() []string                                      // cc: ["plan","default","auto","bypassPermissions"]
    SetMode(session *AgentSession, mode string) error
}

type AgentEvent struct {
    UUID, ParentUUID string
    EventType        string             // cc 用 "user-message"/"assistant-message"/"tool-use"/...
    Payload          json.RawMessage    // EventType-specific
    Timestamp        time.Time
}
```

**v0.1 性质**：
- 单一实现 `ClaudeCodeProvider`，不暴露给 CLI 用户（无 `--agent` flag）
- daemon 不解释 event 内容，原样转发给 wire 层
- wire envelope `output.agent-event` plaintext 含 `providerType` 字段，**v0.1 恒等 `"claude-code"`**——保留字段为 v0.2 留路，但不承诺多 provider 能力
- **接口形态在第二个 provider 实际实作时再稳定**——v0.1 阶段可能因为只看 cc 一种行为有偏差，等 codex/opencode 真要进来时按需 refactor

**v0.1 CLI 表面**（与 D-17 一致）：
- `tether spawn [path]` —— 隐式使用 ClaudeCodeProvider，**不暴露 `--agent` flag**
- **不存在** `tether agents` 命令——v0.1 不承诺多 provider 能力，CLI 不展示 provider 概念
- v0.2+ 真要加多 provider 时再决定 CLI surface（详见 §11.W.3）

### 5.1 ClaudeCodeProvider：与 cc 的三个交互面

cc 进程同时维护**三个独立的 I/O 通道**，互不干扰：

```
┌───────────── cc 进程 ─────────────┐
│                                   │
│  ① stdout/stderr ─────────────────┼──→ PTY master (daemon 持有)
│       (TUI 字节流、ANSI escape)   │     ← daemon 通过 fd 读
│                                   │
│  ② settings.json hook 回调 ───────┼──→ HTTP POST 到 127.0.0.1:<daemon-port>
│       (5 类关键时机)              │     ← daemon 自起 HTTP 端接收
│                                   │
│  ③ ~/.claude/projects/.../<sid>.jsonl
│       (结构化对话历史，cc 自己写) │     ← daemon 通过 fsnotify 增量读
│                                   │
└───────────────────────────────────┘
```

**关键性质**：这三个通道是 cc 这个**应用**的内置行为，跟"cc 怎么被启动"（PTY / SDK / pipe）无关。我们 D-05 / D-06 锁了 PTY 启动方式，但 ② 和 ③ 仍然完整可用。

| 面 | 用途 | 协议 |
|---|---|---|
| ① TTY (PTY) | 字节级输入 / TUI 字节输出 | PTY master ↔ slave |
| ② Hooks | 拦截 PreToolUse / PostToolUse / SessionStart / UserPromptSubmit / Stop 等关键时刻（实时 + 可拦截 + 结构化） | cc 通过 settings.json 配置回调 → 给 daemon 的本地 HTTP 端发 POST |
| ③ JSONL | 结构化对话历史（最终持久 + 完整 + 可 replay） | cc 自己写 `~/.claude/projects/<hash>/<sid>.jsonl` |

**daemon 的视角**：

| daemon 给 Mobile App 的 wire 数据 | 来源通道 |
|---|---|
| `output.agent-bytes`（TUI 字节流，**v0.1 不上 WT**——见 §3.3.3） | ① PTY |
| `output.hook-event`（PreToolUse 等审批触发） | ② Hook HTTP 回调 |
| `output.agent-event`（user / assistant 消息 + tool_use + tool_result）| ③ JSONL 文件 watcher |

**Mobile App 只渲染原生 chat UI**，所以只消费 ② + ③。① 的 PTY 字节流仅本地 `tether attach`（Unix socket）使用。

**为什么 PTY 模式 ≠ "App 拿不到结构化数据"**：cc 在 PTY 模式下仍然写 JSONL（这是 cc 应用层 feature，不是 SDK 模式独有）。daemon fsnotify 监听 JSONL 增量解析就行。**这跟 happy 的 local 模式做法一致**（参考 happy-cli `claudeLocal.ts`：spawn cc 子进程 + watcher JSONL）。Happy 的 remote 模式才走 SDK in-process iterator（D-05 已拒绝）。

### 5.2 daemon 的工作分解

```
spawn cc:
  cmd := exec.Command("claude",
    "--session-id", sessionID,
    "--permission-mode", mode,
    "--settings", customSettingsPath,  // 写好 hook 配置
  )
  cmd.Env = append(os.Environ(), "TERM=xterm-256color", "COLORTERM=truecolor")
  cmd.Dir = sessionCwd
  ptmx, _ := pty.Start(cmd)

PTY 输入（手机/终端 attach 写过来的）：
  Mobile App 消息 → daemon → 清洗 → ptmx.Write([]byte(text + "\r"))   // F-01: CR not LF
  终端按键 → tether-attach → daemon → ptmx.Write(rawBytes)
  (取消 turn 的特殊序列见 §5.3 / F-07)

PTY 输出（fan-out）：
  go func() {
    buf := make([]byte, 4096)
    for {
      n, _ := ptmx.Read(buf)
      ringBuffer.Write(buf[:n])
      for _, c := range attachedClients { c.SendBytes(buf[:n]) }
    }
  }()

JSONL 监听（增量解析三类事件，F-04 / F-09）：
  watcher := fsnotify.NewWatcher()
  watcher.Add(jsonlPath)
  for ev := range watcher.Events {
    if ev.Op & Write != 0 {
      // F-09 防御：仅处理结尾 \n 完整的内容
      newLines := readUntilLastNewline(&offset)
      for _, line := range newLines {
        rec := parseClaudeRecord(line)
        switch classify(rec) {        // F-04 三分类
          case EVENT:  // type=user / assistant，有 UUID
            env := mapToSessionEnvelope(rec)  // ~300-500 LOC mapper
            broadcast(env)
          case HOOK:   // type=attachment + hookEvent
            updateHookState(rec)
          case STATE:  // permission-mode / last-prompt / file-history-snapshot
            updateSessionState(rec)
        }
      }
    }
  }

Daemon 本地 HTTP server（127.0.0.1:0；分两个 namespace）：
  // ── /hooks/* — cc hook 回调（F-05 / F-06）──
  http.HandleFunc("/hooks/pre-tool-use", handlePreTool)        // 拦截 tool 调用
  http.HandleFunc("/hooks/post-tool-use", handlePostTool)      // 拿 tool 输出 + duration_ms
  http.HandleFunc("/hooks/user-prompt-submit", handlePrompt)   // 用户消息直接入手
  http.HandleFunc("/hooks/session-start", handleSessionStart)  // cc 起来了
  http.HandleFunc("/hooks/stop", handleStop)                   // turn 完成

  // ── /blob/* — v0.2 Phase 1 预留命名空间 (D-18) ──
  // POST /blob/register   skill 注册本地文件 → 返回 tether://blob/<sha256>
  // GET  /blob/<sha256>   daemon 内部用（v0.2 Phase 1 启用）
  // v0.1 不实现，但保留 path prefix 不让其他 handler 占
  http.HandleFunc("/blob/", handleBlobNotImplemented)  // 返回 501

  http.ListenAndServe("127.0.0.1:0", nil)  // 端口写进 cc settings + 暴露给 skill

  // PreToolUse 响应 schema（F-05）：
  // {"hookSpecificOutput":{"hookEventName":"PreToolUse",
  //   "permissionDecision":"allow"|"deny"|"ask",
  //   "permissionDecisionReason":"..."}}
  //
  // ⚠️ Hook 是"对话框 pre-answer"通道，不是主防线。
  //   主防线是 cc 的 permission_mode（F-10/F-11）。详见 §5.5。
```

**Hook + JSONL 是两条独立可见通道**：
- Hook：实时、结构化、可拦截（Mobile App 审批 / daemon 决策）
- JSONL：持久、完整、可 replay（断线重连 / daemon 重启）

两条互补：daemon 用 hook 做实时控制流，用 JSONL 做事实派发。

### 5.3 Mobile App 写 cc 的边界处理

Mobile App 发的消息是"用户文本"，daemon 写到 PTY 时要做清洗 + 编码。**以下规则均经 PoC-1 实测验证**（见 Appendix B）：

| 边界 | 处理 | PoC 来源 |
|---|---|---|
| 普通文本 | `text + "\r"` 直接 write（**Submit 是 CR `\r` 不是 LF `\n`**）| F-01 |
| 含 `\r` `\x1b` 等控制字符 | 清洗：保留可打印；换行用 `\r`；其他控制字符 escape 掉 | F-01 |
| 新目录首次启动 | spawn 后等 ~3s，发 `\r` 自动接受 trust prompt（cc 默认选 "Yes, I trust this folder"）| F-02 |
| 长 paste（>1KB / 多行）| 用 bracketed paste 包：`\x1b[200~ + content + \x1b[201~` | 推论（cc 实测启用 `?2004h`）|
| Slash 命令（`/clear`、`/compact`）| 直接 write `"/clear\r"` | F-01 |
| Multi-line 用户消息 | cc TUI 用 Shift+Enter 换行；daemon 注入时把 `\n` 替换成等价 multi-line escape，最后才发 `\r` 提交 | 详见 §11.F 待补 PoC |

实现：~50 LOC `sanitizeAndFormat(text) []byte` 函数。

#### "取消 in-flight turn"序列（F-07）

Mobile App 用户点"停止"按钮时，daemon 执行：

```
1. ptmx.Write(`\x1b`)           // Esc → cc 取消 generation
2. sleep 500–800ms              // cc 把原 prompt 放回输入框 + 渲染稳定
3. ptmx.Write(`\x15`)           // Ctrl+U → 清空输入行
4. sleep 200–500ms
5. (可选) ptmx.Write(newPrompt)
6. (可选) ptmx.Write(`\r`)      // 提交新 prompt
```

总耗时 < 3 秒。**禁止用 SIGINT**：会让 cc 进入僵尸态（TUI 假活、stdin 不响应；F-07）。SIGTERM 才是正确的"杀 cc"路径。

#### Trust prompt 自动化的几种方案（F-02）

`/tmp/foo`（cc 没见过的目录）首次启动时弹 "Yes I trust / No exit" 对话框。daemon 三种处理：

| 方案 | 适用 | 说明 |
|---|---|---|
| 启动后发 `\r` 自动接受 | v0.1 默认 | 简单，但每个新目录都触发 |
| `--add-dir <path>` 启动 | 已知目录 | spawn 时预登记 trust |
| `permissions.allow` settings | 多目录场景 | 全局/项目 settings 写白名单 |

### 5.4 cc 集成层文件清单

参考 happy-cli 的 cc 集成代码，port / 改写到 Go：

| happy-cli 文件 | 处理 |
|---|---|
| `claudeLocal.ts` | port 到 Go：`internal/cc/spawn.go`，但去掉 cross-spawn 等依赖，纯 `os/exec` + `creack/pty`。~150 LOC |
| `generateHookSettings.ts` | port：`internal/cc/hooks.go`，写 cc 的 settings.json with hook URLs。~100 LOC |
| `startHookServer.ts` | port：`internal/cc/hookserver.go`，标准库 net/http。~150 LOC |
| `claudeFindLastSession.ts` | port：`internal/cc/findsession.go`，纯文件遍历。~50 LOC |
| `claudeCheckSession.ts` | port：`internal/cc/checksession.go`，~30 LOC |
| `path.ts` | port：`internal/cc/path.go`，**PoC 实测编码规则**：`/x/y/z` → `-x-y-z`（`/` 全替换为 `-`）。~30 LOC（F-03） |
| `session.ts` | 大幅改写：不复用 happy 的 daemon 协议，session state 自定义。~200 LOC |
| `specialCommands.ts` | port：`internal/cc/specialcmd.go`，~100 LOC |
| `api/encryption.ts` | **不 port**，§11.C 重新定 |
| **新增**：`internal/cc/jsonl_mapper.go` | 把 cc 的 JSONL 记录转成本设计的 wire envelope。~300-500 LOC |
| **新增**：`internal/cc/version.go` | 启动时 `claude --version` 比对 supported range，hard-fail | ~50 LOC |

**Go cc 集成层总量**：~1100-1400 LOC，因为不再处理 SDK 转换层。

### 5.5 cc 的三层权限模型（PoC-1 实测发现）

这是 PoC-1 最重要的发现之一（F-10 / F-11），**直接简化了原 §11.D 的 lock 设计**：daemon 不需要造任何"权限锁"，cc 自己已经分了三层。

```
                    cc 三层权限模型
              ════════════════════════════

                user prompt → assistant 决策
                          │
                          ▼
        ┌─ Layer 1: Model-level（plan mode）─┐
        │ system prompt 告诉 LLM "plan mode" │
        │ LLM 自己拒绝调用 tool              │
        │ → 0 hook 调用、0 tool_use         │
        │ ✅ 真 fail-closed（含 echo）       │
        └─────────────┬──────────────────────┘
                      │ (其他 mode)
                      ▼
        ┌─ Layer 2: Built-in safe-cmd ──────┐
        │ cc 内置无副作用命令 allowlist     │
        │ （PoC 实测 echo 在内；ls/cat/pwd  │
        │  推断在内但需个案验证）           │
        │ → 自动放行（不走 hook、不走对话框）│
        │ ⚠️ hook 失败也放行（fail-open）   │
        └─────────────┬──────────────────────┘
                      │ (非 safe 命令)
                      ▼
        ┌─ Layer 3: mode + Hook + TUI ──────┐
        │ default mode：TUI 弹对话框         │
        │   ├─ Hook 可 pre-answer 对话框    │
        │   └─ Hook 失败 → 等用户在 TUI 按键  │
        │     ✅ 危险命令 fail-closed        │
        │ auto / acceptEdits / bypass：     │
        │   各种程度自动放行                  │
        └────────────────────────────────────┘
```

**PoC-1 实测矩阵**（F-10/F-11，见 Appendix B 详情）：

| permission_mode | echo（safe） | touch（side-effect）| hook 是否调用 |
|---|---|---|---|
| `plan` | **拒绝**（model-level）| **拒绝** | **0 次** |
| `default` | **执行**（safe-cmd allowlist）| 弹 TUI 对话框等响应；hook timeout → 不执行 | 调用（仅 Layer 3）|
| `auto`（cc 默认）| 执行 | 执行（hook 任何故障都 fail-open）| 调用 |

#### 设计推论：tether 不需要造"权限锁"

```
tether daemon 的策略 = 顺水推舟用 cc 已有的 3 层
─────────────────────────────────────────────

启动默认 permission_mode = default
              │
              │  echo（实测）等内置 safe-cmd 直接跑（Layer 2 自动放行）
              │  touch/rm 等有副作用命令 → hook 调用 → 手机 Mobile App 决策
              │  daemon 死 → hook 失败 → 危险命令的 TUI 对话框停在那里等人
              │  → 用户用 `tether attach` 进终端 → 看到对话框 → 按 Y/N
              │
Mobile App 通过 RPC 切 mode：
              ├─ "切到 auto / acceptEdits"  → 信任此会话，自动跑
              ├─ "切到 plan"                → 只允许规划，不能动 tool
              └─ "切到 default"              → 默认值
```

**Lock state machine 仅管"谁能写 PTY 字节"**——permission_mode 由 cc 自己分层处理。详见简化后的 §11.D。

### 5.6 JSONL 解析与防御（F-04 / F-09）

把 cc JSONL 文件流转换成 daemon 可用事件流的具体规则。这是 daemon 的"事实派发"管道（与 hook 的"控制流"互补，见 §5.2 末尾）。

#### 三类分类（F-04）

```
对每条 JSONL 行 → switch on classify(record):

  EVENT  →  type = user / assistant
            有 `uuid`，构成消息父子链
            映射成 wire envelope 广播给 Mobile App

  HOOK   →  type = attachment + attachment.hookEvent 非空
            元数据：哪个 hook 在什么时机触发了什么
            更新 daemon 内部 hook 状态机

  STATE  →  type = permission-mode / last-prompt / file-history-snapshot
            无 `uuid`
            更新 session 状态视图（不广播）
```

#### Torn-write 防御（F-09）

PoC-1 step 9 实测：1ms 频率轮询 × 45s × 记录最大 10.6KB，0 次 torn write 命中。结论：cc 用 buffered I/O 把整条 record 凑齐再 flush，不是逐字节追加。

**但 daemon 仍必须做防御**（防 1ms 盲点 + 未来 cc 版本变化）：

- 读取后**仅处理结尾是 `\n` 的内容**
- 末尾不是 `\n` → 视为"写到一半，等下次事件"，不 advance cursor
- UUID 索引按行 uuid 去重（即便 cc 重试导致同一行写两次也不重复广播）
- daemon 重启时全量重建 UUID 索引（read-only 扫一遍 JSONL）

实现：~80 LOC `jsonl.IncrementalReader`。

---

## 6. Daemon 死亡 / 恢复契约

### 6.1 持久化清单

| 状态 | 是否落盘 | 位置 |
|---|---|---|
| Session 注册表（id, **providerType**, cwd, **workspaceRoot**, **mode**, createdAt, attached devices） | ✅ 落盘（每次 `session-set-mode` RPC 后**原子刷新**，否则 daemon 崩重启 agent 会用错 mode）。**`workspaceRoot`** 字段为 v0.2 dag skill 找 `<workspace>/.tether/` 用，默认 = cwd，spawn 可显式指定 | `~/.tether/users/default/sessions/<id>/meta.json` |
| 对话历史 | ✅ 落盘（每个 provider 自己负责持久化）| ClaudeCodeProvider: `~/.claude/projects/<hash>/<sid>.jsonl`（cc 自己写）；其他 provider 各自定义 |
| 设备配对、token、device pair wrap_key | ✅ 落盘 | `~/.tether/users/default/devices.json`、`keys.json` |
| **`session_key`**（每 cc session 一个，跟 cc session 等长，**不**是 daemon process 等长）| ✅ 落盘（用 `wrap_key` 加密后落地；cc session 真正结束时清除）| `~/.tether/users/default/sessions/<id>/keys.bin` |
| Lock 状态（当前 holder） | ❌ 重启归零 | 内存 |
| 进行中 tool 审批 | ❌ 重启归零 | 内存 |
| PTY ring buffer | ❌ 重启归零 | 内存（cc 重启会 redraw） |
| Mobile App 的 `lastSeenEventUuid` | ✅ 落盘（Tauri FS plugin → 应用沙箱目录） | App 侧 |

### 6.2 daemon 启动恢复流程

```
tether daemon 启动
├─ 读 ~/.tether/users/default/sessions/*/meta.json
├─ 对每个 session:
│   ├─ kill -0 <pid>?
│   │   ├─ 活着（罕见）→ 标 "orphan"，等用户决定（killed/adopted）
│   │   └─ 死了（常见）→ 标 "needs-resume"
│   └─ 重建 JSONL UUID 索引（read-only，不修改 cc 文件）
├─ 起 Unix socket listener（local control，给 `tether attach` 用）
├─ 起 client goroutine（QUIC outbound 连 server）
└─ 等 client / attach 连进来
```

JSONL 增量解析的具体规则（防御 + 三类分类）见 §5.6。

### 6.3 用户视角

**Mobile App 重连后**：
- 看到 `session-state { status: "needs-resume" }`
- UI banner：`会话中断，点击恢复 →`
- 点 → `session-resume` RPC → daemon spawn 新 cc with `--session-id <uuid>`
- cc 启动 1-3s → JSONL 历史增量同步给 Mobile App → 用户继续

**终端 attach**：
- `tether attach <sid>`
- 默认 `--auto-resume`：daemon 自动 spawn cc，attach 进入
- `--no-auto-resume`：daemon 提示先 `tether resume <id>`

**显式 `tether resume <id>`**：
- 用于"想 resume 但暂时不 attach"
- 同 Mobile App resume 流程

### 6.4 真正会丢的东西

明确告诉用户的 caveat：

1. **进行中的一个 turn**：daemon 死的瞬间正在生成的 turn 整体丢失（cc 在 turn 完成前不写完整 JSONL，所以用户重发就好）
2. **未审批的 tool 调用**：手机上正在弹的"是否允许"在 resume 后不会重弹（turn 整体作废）
3. **窗口期内 Mobile App 发的消息**：靠 App 端本地暂存（Tauri FS queue，~50 LOC），daemon resume 后自动 replay

### 6.5 速度目标

- daemon 进程启动：< 1s
- cc spawn + JSONL load + TUI redraw：1-3s
- 用户感知总停机：5-10s（含手动点 resume 的反应时间）
- 通勤场景预估 频次：≤ 每天 1 次（依赖 daemon 稳定性，目标 1 周 1 次以下）

### 6.6 三类恢复路径（P1-6 review 加）

`tether doctor` 自检后输出四类状态，对应不同恢复动作：

| 状态 | 触发条件 | 恢复动作 |
|---|---|---|
| **健康** | meta.json + keys.bin + devices.json + cc JSONL 全部一致 | 无需操作 |
| **可 resume** | meta.json 标 needs-resume，keys.bin 完整 | `tether resume <id>` 重 spawn cc + 加载 session_key（§6.2）|
| **需重新配对**（部分） | devices.json 损坏 / device key 丢失 / Keystore 被 OS 重置 | `tether revoke <deviceId> && tether pair`：仅重配对受影响 device，其他 device + 其他 session 不受影响 |
| **只能新建 session** | keys.bin 永久丢失（磁盘故障 / 用户误删 / 备份不当）| `tether kill <id>` + `tether spawn`：旧 session 历史**永久不可恢复**（server 暂存的 ciphertext 无 key 解，cc JSONL 文件本身仍在但不被 tether 跟踪）|

**关键性质**：每类恢复**互不影响**——一个 session 的 keys.bin 丢不会污染其他 session；一个 device 被 revoke 不影响其他 device；daemon 二进制崩溃不影响磁盘状态文件完整性（atomic write 保证）。

**`tether doctor --fix` 行为**：
- 健康 / 可 resume：no-op 或自动 trigger resume
- 需重新配对：交互式提示用户扫 QR
- 只能新建 session：交互式确认后 archive 旧 meta.json + keys.bin（不删，移到 `.archive/<ts>/`）

---

## 7. 切换心智模型

### 7.1 cc 进程从不重启

跟 happy 的根本区别：

| 切换瞬间 | Happy | 本设计 |
|---|---|---|
| cc 进程 | 杀掉重启（local binary ↔ remote SDK 互切） | 同一个 cc 不动 |
| 进程切换耗时 | spawn + JSONL resume，~几百 ms 到 ~2s | 0（只 lock 翻转） |
| 用户体感 | 屏幕闪一下，TUI 重画 | 完全无感 |
| 中间字节风险 | outgoing message queue 要 flush | 无切换边界 |
| daemon 死的代价 | cc 通常没事（cc 是用户终端的子进程） | cc 死，需要 resume |

**优势**：物理上无缝。
**代价**：D-07 那个 daemon-死风险（已接受）。

### 7.2 Phone → PC 流程

```
T0：通勤中，Mobile App 在驱动
   Mobile App → WebTransport stream → server → CLI client goroutine → fromServer chan
   → daemon → input gate (lock=mobile-A) → ptmx.Write → cc 吃字节
   cc 写 JSONL → daemon JSONL watcher → 上行 envelope → server → Mobile App

T1：你 SSH 到 GCP VM，跑 `tether attach <sid>`
   tether attach 进程启动 → term.MakeRaw(stdin) → net.Dial(daemon's Unix socket)
   daemon 把 ring buffer 发回 → 立刻看到 cc 最后一屏 TUI

T2：你按下任意键
   keystroke → io.Copy → daemon Unix socket → input gate
   ├─ holder=mobile-A, lastWrite < 60s ago
   │   → 拒收 → control msg → 终端覆盖 banner（"Mobile holds lock. Ctrl+\\ Ctrl+T to force"）
   └─ lastWrite > 60s ago
       → auto release → holder=terminal-B → 字节 → ptmx → cc

T3：你按 Ctrl+\\ Ctrl+T 强制接管
   tether attach 识别 → 发 TakeoverRequest control msg
   daemon flip holder=terminal-B → emit session-state 给 Mobile App
   Mobile App 输入框灰掉 "Terminal is active"

T4：你正常打字，cc 同一个进程响应
   Mobile App read-only 看着输出
```

### 7.3 PC → Phone 流程

```
T0：你在终端打字
T1：合上电脑（SSH 断）/ Ctrl+\\ detach
   attach 客户端断 → daemon 知道 client 走了
   60s 内无 input → daemon auto holder=null

T2：通勤路上拿手机，Mobile App 通过 QUIC connection migration 自动重连
   （手机从 WiFi 切到 4G，Connection ID 不变 → 完全无感）
   Mobile App 看 holder=null → 用户打字 → "继续刚才的重构"
   消息走 server → CLI client goroutine → daemon

T3：daemon 收到
   input gate: holder=null → 接受 → holder=mobile-A
   ptmx.Write([]byte("继续刚才的重构\\r"))  ← 关键（Submit 用 CR，F-01）
   cc 收到 stdin → TUI 渲染输入框 → submit

T4：cc 思考 + 写 JSONL → daemon watcher → server → Mobile App
```

### 7.4 QUIC connection migration 的天然 bonus

- 手机 WiFi → 4G 漂移：QUIC 用 Connection ID 而非 4-tuple，连接迁移**对应用层透明**
- 笔记本休眠唤醒：同上
- 不需要 Mobile App 端做特殊重连逻辑——QUIC connection migration 直接处理

---

## 8. Lock 状态机（骨架）

> **PoC-1 后简化**：原本担心要叠加"permission_mode 锁"层，§5.5 实测后明确不需要——cc 自己分了三层权限，daemon 的 lock 只管"谁能写 PTY 字节"。详细规则见 §11.D（已收敛）。

### 8.1 状态

```go
type Lock struct {
    Holder        *ClientID    // nil = 无 holder
    LastWriteAt   time.Time
    HistoryLog    []TakeoverEvent
}

type ClientID struct {
    Kind     string    // "mobile" | "terminal"
    DeviceID string
}
```

### 8.2 规则（细节见 §11.D）

- **终端 attach 隐式拿 lock**：第一个字节按下时，如果 holder=null → 自动拿；如果 holder=mobile + idle > 60s → 自动接管；如果 holder=mobile + active → 拒收 + banner
- **Mobile 显式请求 lock**：Mobile App 发 `lock-request`，规则同上
- **Force takeover**：双向都支持
  - 终端：`Ctrl+\\ Ctrl+T`
  - Mobile App：UI 按钮 "Take over"
- **自动释放**：60s 无 input → holder=null
- **Audit log**：每次状态变化写 `~/.tether/users/default/sessions/<id>/lock.log`

---

## 9. Server 角色（v0.1 范围）

### 9.1 Server 必须做的事

| 职责 | 描述 |
|---|---|
| QUIC listener for CLI | 接受 CLI 的 outbound 连接，鉴权后挂在 user 名下 |
| WebTransport listener for Mobile | 接受 Mobile App 的连接（CLI 也走同一个 listener，wire 对称），鉴权后挂在 user 名下 |
| Auth & 配对 | 公钥签名挑战 → 颁发 access token；Mobile-CLI 配对走 server 中转 |
| Envelope 路由 | Mobile 发来的 envelope 转发到该 user 的 CLI；CLI 发来的转发到该 user 所有在线 Mobile App |
| 离线消息暂存 | CLI 离线时，Mobile 发的消息存在 server，CLI 上线后增量推送（B4 + 1h TTL，§11.B） |
| ~~Push 通知~~ | **不参与**：CLI 自己直推 APNs/FCM（E2，§11.E）|
| 不解密 | 内容字段是不透明 base64 ciphertext（前提是 §11.C 决定 E2E） |

### 9.2 Server 不做的事（v0.1）

- Multi-user（D-04 锁定）
- Multi-machine（同上）
- 文件存储 / artifacts
- 数据库（SQLite 起步够用，§11.H 待定）
- 跨地域 / 高可用（单实例就行）

### 9.3 部署形态（§11.I 待定）

两个候选：
- **A. server + CLI 同机**：在你 GCP VM 上 `tether-server` + `tether daemon` 都跑，Mobile App 通过 GCP 的公网 IP 连
- **B. server 独立**：fly.io / Railway / Cloudflare Workers + Durable Objects 跑 server；CLI 跑你笔记本；两边都 outbound 到 server

---

## 10. PoC 优先级（建议）

讨论的 Bet 1（PoC #3 daemon-PTY 死活问题）已经被 D-07 接受，不再 PoC。

新的 PoC 优先级建议（按风险高低）：

### PoC-1 ✅ **已完成**（2026-04-27，12 步全过）

**实际跑了 12 步**（超出原计划 6 项）：
1. ✅ Go 起 cc + PTY 字节流（true-color / bracketed paste / focus event 都开）
2. ✅ PTY 写入 + cc 处理（**关键发现**：Submit 是 `\r` 不是 `\n`，F-01）
3. ✅ JSONL fsnotify 增量解析 + 三类分类（F-04）
4. ✅ Hook PreToolUse allow/deny 双向生效（F-05）
5. ✅ 5 类 hook 完整事件流：SessionStart / UserPromptSubmit / PreToolUse / PostToolUse / Stop（F-06）
6. ✅ Turn 中断：Esc 清洁取消、SIGINT 让 cc 进入僵尸态（F-07）
7. ✅ Esc + Ctrl+U 清输入框后新 prompt 不拼接（F-07）
8. ✅ 多 session 100% 隔离 + 单 hook server + session_id 路由（F-08）
9. ✅ JSONL 实测 0 torn write，但 daemon 仍要"末尾非 `\n` 视为未完整"防御（F-09）
10. ✅ Hook 故障下 cc 行为：safe-cmd 自动放行；非 safe 走 TUI 对话框（F-10）
11. ✅ `--permission-mode default` 对危险命令 fail-closed（TUI 对话框等用户）（F-10）
12. ✅ `--permission-mode plan` 对所有命令包括 echo fail-closed（model-level 拒绝）（F-11）

**通过标准**：12 项全过 + 11 条事实见 Appendix B。**实际耗时 1 个工作日**（2026-04-27）。

代码：`poc/go-pty-cc/` ~1900 LOC Go。

### PoC-2（进行中，week 1）：WebTransport 双向通信 — **HARD GATE**

**v0.1 transport 栈硬验证点**——失败必须立刻降级，不能"边开发边修补"。

**验证**（截至 2026-04-28，**5/6 项已 PASS**——§11.T HARD GATE 实质性通过；step 4 是真机 + 网络切换的外部验证，不阻塞实现）：
1. ✅ **PASS** Go server 起 WebTransport-over-HTTP/3 单 listener（`poc/go-quic-wt/step1_server.go`）
2. ✅ **PASS** Go CLI client 用 webtransport-go client mode 连 server，发收 envelope（`step2_client.go`，验 §11.T 风险点的基础形态）
3. ✅ **PASS** Browser ↔ Go server 互通（`step3_browser/index.html` + `step3_static.go`，**desktop Chrome 跨公网到 GCP VM 跑通 5 项 ✓**；**Android Chromium 真机也 PASS**——通过 Cloudflare quick tunnel 提供 HTTPS secure context 让 HTML 加载，WT 直连 GCP VM:4433。直接验证 D-13 Tauri Android only 决议的核心可行性）
4. ⏳ **TODO** 跨网络切换（4G → WiFi）connection migration 验证（真机 + 网络切换；外部环境验证，不阻塞实现）
5. ✅ **PASS** 多 stream 并行（control / events / agent-bytes / catchup 4 bidi stream + datagram heartbeat 不互相阻塞）（`step5_streams.go`，localhost 实测见 Appendix C）
6. ✅ **PASS** CLI 长连接 30 分钟 + 心跳，0 leak / 0 reconnect / 99.9% datagram（`step6_longrun.go`，2026-04-28 实测见 Appendix C）

**已捕获的 webtransport-go v0.10 行为事实**（4 条 P-XX，详见 Appendix C）：
- **P-01** `webtransport.ConfigureHTTP3Server(h3)` 必须显式调，否则 client 报 "server didn't enable WebTransport"
- **P-02** `EnableStreamResetPartialDelivery: true` server 和 client 端的 `quic.Config` **都必须**显式开
- **P-03** Datagram **两层都开**：`http3.Server.EnableDatagrams = true` + `quic.Config.EnableDatagrams = true`
- **P-04** 客户端 `tls.Config.NextProtos` 必含 `"h3"`

**通过标准（HARD GATE）**：6 项全过 + connection migration 在 Android demo 上 < 1s 切换无丢包 + CLI 长连接无 leak。**任一项失败立即触发降级，不进入实现阶段**。

**当前 §11.T 风险点状态**（截至 2026-04-28，**HARD GATE 实质性通过**）：
- ✅ webtransport-go client mode 基础稳定性（step 2）
- ✅ 多 stream 并行 + datagram + 大 blob 不堵小消息（step 5）
- ✅ **30 分钟长连接 + 心跳 + 0 leak**（step 6 实测：1MB heap 不变、0 panic、0 reconnect、ctrl/evt 0 失败、datagram 99.9%）
- ✅ **Browser ↔ Go server 互通**（step 3 desktop Chrome 跨公网 GCP VM 全 5 项 ✓——dial / bidi / datagram / multi-stream / close）
- ⏳ Connection migration 真机（step 4，外部环境验证，不阻塞实现）

**§11.T 核心风险全部消除**——webtransport-go v0.10 在 server + Go client + Chrome browser 三方互通已验证，长连接 30min 稳定。剩 step 4 connection migration 是真机 + 实际 4G/WiFi 切换的边界验证，不阻塞实现启动。

**降级路径**（按风险递增）：
- 第 2 / 6 项失败（webtransport-go client mode 不稳）→ **CLI 改走 raw QUIC HTTP/3**（quic-go 直接用，自定义 framing），server 同 listener 加 raw QUIC route
- 第 3 项失败（Android WT dial 不稳）→ 已在 P1 #5 = ii 决议下提前选 quinn polyfill；这条降级 v0.1 不再适用（Android 已经走 polyfill）
- 第 4 项失败（migration 不可靠）→ App / CLI 加显式重连 + session.resume 协议层补偿
- **绝不引入 WebSocket / TCP fallback**——跟 D-02 / §11.A 一致；UDP/443 不通靠 §11.K 的 Tailscale Funnel / Cloudflare Tunnel 解决

### PoC-2.6（必做，Tauri build matrix commit 之前）：`web-transport-quinn` × Android smoke test

**目的**（review 2026-04-28 P1 #6 + 锁定 P1 #5 = ii-rebuilt 统一 homebrew polyfill 的前置依赖）：在 v0.1 commit 1.5 周 Tauri build matrix 工作之前，验证 `web-transport-quinn` 这个 Rust crate 在 Android target 编译 + 跑通基本 WT dial。

**为什么仅 Android smoke**（vs 之前计划的"四平台 spike"）：
- 桌面（macOS / Linux / Windows）：quinn / web-transport-quinn 在桌面是常规 Rust 编译，无 Tauri 特殊 glue，开发期写代码一边验，不需要专门 spike
- iOS：v0.1 不在范围（D-13 推迟 v0.1.x）；v0.1.x 启动时另起 iOS smoke
- **Android**：唯一未知项——需要 Kotlin / JNI glue 接 Tauri Mobile 的 Rust 端 + UDP socket 权限 + `aws-lc-rs` / `ring` crypto provider 在 `aarch64-linux-android` 编译

**验证步骤**：
1. **本地桌面 macOS smoke**（开发者主机，作为 baseline）：Tauri 2.10.x desktop project + 自写 Tauri command 包 `web-transport-quinn` → 连 PoC-2 step 1 server (`bin-step1`) → 跑 PoC-2 step 3 browser test 等价 5 项（session ready / bidi echo / datagram echo / multi-stream / close clean）
2. **Android emulator**（Android Studio）：Tauri 2 Mobile project + 自写 Tauri command + `aarch64-linux-android` 编译通过 → 同 1 跑 5 项 PASS
3. **真机 Android**（用户已有设备，跨公网 GCP VM）：emulator 通过后真机验，主要确认无非模拟器 bug

**通过标准**：3 个环境全部完成 5 ✓ PASS。

**预算**：1-2 天（写 thin Tauri command + 跑 3 个环境）。如果 Android target 编译 / glue 出问题（最大风险点：Tauri Mobile Android plugin scaffolding 没有 web-transport-quinn 集成案例），暴露在此阶段而非后期。

**降级路径**：
- Android 编译失败（quinn 在 `aarch64-linux-android` 有未知 issue）→ 调研替代 Rust QUIC 实现（`wtransport` 是备选，659 stars），或 Android 临时改走 native WT（Chromium WebView 自带）+ 桌面用我们的 homebrew，timeline +0.5-1 周
- 真机失败但 emulator 通过 → 调研真机网络 / UDP 限制；可能要加重连 / fallback 逻辑（不 block v0.1 发布）

**LOC 影响**：自写 Tauri command 的 ~500-1000 LOC 在 Tauri build matrix 那 1.5 周里完成（Android smoke 用早期版本，build matrix 期完整化）。

### PoC-2.5（必做，跟 PoC-2 同期）：PTY ring buffer overflow stress

**验证 §3.3.4 L1 ringbuf 4MB + 丢老 agent-bytes 行为**（P1-9 review）：
1. cc 一次性输出 5MB（如 `cat large_file`）→ daemon ring 不阻塞 cc stdin，PTY reader 持续 drain
2. cc 一次性输出 50MB → 同上 + ring overflow 时丢老 agent-bytes 字节（保 events stream 上的 hook/jsonl）+ emit `control.error{kind:"ringbuf-overflow"}` 给 App
3. cc 长时间慢节奏输出（每秒 5KB × 30 分钟）→ ring 平稳维持
4. App 慢消费（人为 100ms 延迟收 envelope）+ cc 大段输出 → server 端 trim cc-bytes 触发 catch-up

**通过标准**：cc 进程在所有压力场景下 stdin 不阻塞、不 hang；events stream（jsonl/hook）不丢；overflow 只丢可丢的 agent-bytes 字节流；client 收 catchup-needed 后能恢复一致性。

### PoC-3（必做，week 1-2）：watchdog 模式实测
**验证**：
1. 注入 panic 到 daemon goroutine，cc 是否存活、新 daemon goroutine 是否正常接管
2. 注入死循环到 daemon，watchdog 是否在 10s 内发现 + cancel ctx + 重启
3. main 进程被 SIGKILL，下次启动 needs-resume 流程是否正常

**通过标准**：1-2 项 cc 完全无感；3 项 resume 在 5-10s 完成且 Mobile App 正确显示恢复中。

### PoC-4（可选，week 2-3）：JSONL 增量 replay 在网络异常下的正确性
**已部分验证**：F-09 实测 0 torn write 命中（轻量场景）。下面是更全面的随机崩溃测试。

**仍要验证**：
1. Mobile App 离线 5 分钟，期间 cc 正常工作
2. Mobile App 上线，按 `lastSeenEventUuid` 增量 sync
3. 中间不丢、不重复、顺序正确
4. 模拟人为 torn write（中断 cc）时 daemon 防御逻辑是否被正确触发

**通过标准**：100 次随机崩溃测试无错。

---

## 11. 待讨论决策清单

每项格式：**问题** / 已知选项 / 已知约束 / 当前倾向。

### 11.A 浏览器侧 transport ✅ **已锁定（纯 WebTransport，不做 WS fallback）**

**决定**：浏览器只走 WebTransport-over-HTTP/3。无 WS fallback、无 transport adapter 抽象——业务代码直接耦合 WebTransport API。

**理由**：
- WS fallback 会迫使 envelope schema 自降到 WT/WS 共同子集，**等于把 QUIC 的 datagrams、多 stream、connection migration 全废掉**——又退回到一开始想避开的那条路
- v0.1 / dogfood 期用户路径（自己 GCP VM、第二个用户朋友）UDP/443 通断已知或可控
- 真有用户网络阻塞 UDP/443，引导他们用 Tailscale Funnel / Cloudflare Tunnel（§11.K）解决，而不是在 client 端写 dual code path
- 留 transport adapter 抽象但不做 WS adapter = 空闲抽象腐烂；干脆不留

**约束**：
- WebTransport 在 Chrome/Edge/Firefox 都已稳定；Safari 26.4+ 支持
- 浏览器没有 raw QUIC API（WebTransport 是唯一接口）

见 D-02（§2）。

### 11.B Server 离线消息暂存策略 ✅ **已锁定（B4 + session_key 跟 cc session 等长 + trim 策略）**

**决定**：
- **server 暂存策略 = B4**：暂存 ciphertext envelope 到 CLI ack；**1h TTL fallback**（防 CLI 永远不回来）；单 session 上限 **10MB**（防 Mobile App bug 灌爆）
- **session_key 持久化跟 cc session 等长**（**不**跟 daemon process 等长）：daemon 重启时从 `~/.tether/users/default/sessions/<id>/keys.bin` 恢复，能解 server 暂存的旧 ciphertext
- **暂存满（10MB）时的 trim 策略**：删老的 `output.agent-bytes` envelope，**保**所有 events / control / input。trim 后给该 client 发 `control.catchup-needed`，触发 stream 4 catch-up（详见 §3.3.4 / D-14）

**为什么 B4 比 B1/B2/B3 好**：
- B1 把责任全推给 Mobile App，但 App 后台被 OS kill 时本地 queue 丢
- B2/B3 让 server 持久化太多——E2E 下 server 看不到内容，无法做有意义的 GC，只能按 size + ID 限流
- B4 是"server 兜底 + Mobile App 端也兜底"的组合：双方都有 retry，但 server 暂存的窗口不长（CLI 一上线就清）

**为什么 session_key 必须跟 cc session 等长**（驱动这条修订的硬约束）：
- C1 锁定后 server 暂存的是 ciphertext，用当时的 `session_key` 加密
- 如果 daemon 重启时 rotate session_key，server 暂存的旧 ciphertext **无法解密**（用新 key 解不了用旧 key 加密的字节）
- 解决：session_key 持久化到磁盘，跟 cc session 等长，仅在 cc session 真正结束（用户 `tether kill <id>`）时 rotate

**Forward secrecy 影响**：
- 原方案："daemon 重启就 rotate" → cc session 内多次 rotate
- 新方案："cc session 结束才 rotate" → 削弱：攻击者拿到 daemon 磁盘 + cc session 还活着 → 能解历史
- 单用户场景威胁可接受；wrap_key 仍在 Keychain 强保护

详见 §6.1 持久化清单。

见 D-12（§2）。

### 11.C E2E 加密层级 ✅ **已锁定（C1 精简版）**

**决定**：v0.1 直接走完整 E2E（C1）。server 是哑中继，看不到 plaintext，只看 metadata（sessionId / deviceId / nonce / opaque ciphertext）。

**驱动因素**：
- 用户路径：dogfood 期 server + CLI 同机 → 稳定后部署 GCP（Google hypervisor，物理边界不再受用户控制）→ 第二用户很快到位
- C2 → C1 的窗口期太短，"先 C2 后升 C1"实质等于写两遍 protocol
- "keyVersion 占位"是 false comfort：v0.2 加 E2E 仍需重设计配对协议，等于破坏性升级
- cc session 内容（代码、secret、内部业务逻辑）比一般 chat 敏感得多

**精简版本**（~390 LOC Go，比 happy 多用户场景的复杂版少一截）：

| 层 | 算法 / 机制 |
|---|---|
| Key exchange | **X25519** ECDH（配对一次性） |
| Symmetric AEAD | **XChaCha20-Poly1305**（24B nonce 全 random，不用持久 counter） |
| Key derivation | **HKDF-SHA256** |
| 密钥层级 | device-pair 长期 wrap_key → 每 session ephemeral session_key（forward secrecy 在 session 边界） |
| Server 看到 | sessionId / fromDeviceId / toDeviceId / nonce / keyVersion / ts / opaque ciphertext |
| Device 撤销 | 默认 A：只吊销该 device，其他不受影响。`tether revoke --rotate-all` 提供高强度选项 |

**密钥派生流程**：

```
配对一次性：
  Mobile App gen (sk_app, pk_app)，存 iOS Keychain / Android Keystore (via tauri-plugin-keyring)
  CLI gen (sk_cli, pk_cli)，存 ~/.tether/users/default/keys.json
  扫 QR 后双方 ECDH(sk_self, pk_peer) → device_pair_secret (32B)
  HKDF(device_pair_secret, "tether-pair-v1") → wrap_key

每个 cc session 一次（forward secrecy）：
  CLI 起 session 时 gen session_key (32B random)
  XChaCha20-Poly1305(wrap_key) 加密 session_key → 一条 control envelope 发给 Mobile App
  cc session 结束 → CLI 持久化清掉 session_key（Mobile App 同步清）

每条 envelope：
  nonce = 24B random
  AD = sessionId || fromDeviceId || toDeviceId || keyVersion
  ciphertext = XChaCha20-Poly1305.Seal(session_key, nonce, payload, AD)
```

**LOC 估算（精简版）**：
- Pairing + ECDH + QR encode/parse: ~120 LOC
- Session key wrap/unwrap + 持久化: ~80 LOC
- Envelope encrypt/decrypt + AD/nonce 管理: ~80 LOC
- 设备表 / 配对状态 / keys.json schema: ~80 LOC
- Dev mode 后门：`TETHER_DEV_DECRYPT=1` 让 server 拿一份 dev key 解日志，build tag 控制（生产 binary 编译时完全剔除）: ~30 LOC
- **合计 ~390 LOC**

**子决策**：
- **配对 transport**：
  - **默认（QR 直扫，强 MITM 抵抗）**：CLI 显示 QR 含 `{cli_pubkey, fingerprint, server_url, pairing_token, server_pubkey_pin}`；Mobile App 用 `tauri-plugin-barcode-scanner` 扫，扫完显示 fingerprint 给用户视觉确认（前 4 字节 + 后 4 字节 hex，配双方设备名）
  - **fallback（server 中转 token，需 SAS 验证）**：双端各显示 6 位 short authentication string（SAS = HKDF(shared_secret, "tether-pair-sas") 取 6 digit），用户必须**双端口头比对一致**才确认配对；不比对的 fallback 路径**仅在 dev/debug binary 启用**，启动横幅 banner 警告
- **Device 撤销**：默认走 A（只吊销该 device，其他不受影响）；`tether revoke --rotate-all` 高强度选项吊销时所有剩余 device 必须重新配对
- **keyVersion 字段**：v0.1 = 1 恒定；真协议升级（换算法）才 bump → 2。envelope 总是加密
- **Dev mode 后门**：`TETHER_DEV_DECRYPT=1` 仅 dev build tag 启用——production binary 编译时该路径**完全删除**（build tag `tether_dev` 不在 release CI 启用）。**CI 必须 grep release binary symbols 确认无 dev decrypt 函数符号**；release binary 启动横幅打印 `crypto-mode: production`（dev build 打印 `crypto-mode: DEV-DECRYPT-ENABLED`）

**Server-compromise confidentiality 保证**（从原"forward secrecy"重命名——P0-5 review 修正）：
- **server 单独被攻击（拿到 SQLite + ciphertext 队列）**：server 看不到任何 plaintext envelope/blob 内容；仅暴露 routing metadata（`kind` / sessionId / deviceId / ts）
- **device 长期 key（wrap_key）泄露**：未来 session 暴露；**已结束的 cc session（session_key 已清掉）历史仍安全**
- **daemon 磁盘冷攻击 + cc session 还活着**：`session_key` 在 `keys.bin` 仍存在 → 攻击者**能解该 cc session 的 server 暂存历史 + 当前 envelope**——这是 §11.B 决议的代价（B4 + session_key 跟 cc session 等长换 daemon 重启可恢复）
- **真正的"消息级 forward secrecy"**（每消息 ratchet, à la Signal）v0.1 不做——场景没必要
- **缓解**：CLI 长期 key 走 OS keychain（macOS Keychain / Linux Secret Service / Windows DPAPI），不裸落盘；`session idle timeout`（v0.1.x 加，cc session 长时间无 input 自动 rotate session_key）；`tether kill <id>` 主动结束并清 session_key

**v0.2 Blob 加密 contract**（D-18 预留，v0.1 不实现，但 wire 形态已定，v0.2 直接照写）：

v0.2 Phase 1 引入 `tether://blob/<sha256>` 二进制 blob 传输（详见 `2026-04-27-phase1-dag-protocol.md` §7）。E2E contract：

- **sha256 是 plaintext 内容的 hash**（daemon 注册时本地计算，不信任 skill；URL 即身份）
- **Wire 传输用 session_key（D-12）加密**，跟 envelope 同一把 key，**通过 AD domain separator 区分**

**Chunk + AEAD 帧格式**（D3 决议）：

```
chunk_size = 256 KB（plaintext）

body = frame_0 || frame_1 || ... || frame_{K-1}    // K = total_chunks

每帧：
  ┌──────────┬──────────┬──────────────────┬────────┐
  │ 4B BE    │ 24B      │  N bytes         │ 16B    │
  │ size N+40│ nonce    │  ciphertext      │ tag    │
  └──────────┴──────────┴──────────────────┴────────┘

AEAD 参数：
  key       = session_key（D-12）
  nonce     = 24B random
  AD        = "tether-blob-v1" || sha256(32B) || chunk_index(8B BE) || total_chunks(8B BE)
  ciphertext = XChaCha20-Poly1305.Seal(key, nonce, plaintext_chunk, AD)
```

**AD 三层防御**：domain separator 防跨域 replay；chunk_index/total_chunks 防重排序+截断；sha256 绑定 blob 身份。

**App 校验流程**：逐帧解密 → 累积 plaintext → 全收后算 SHA256 → 跟 URL 里 sha 比对，不等就拒绝重试。

**Auth**：HTTP/3 GET `/blob/<sha>` 用 `Authorization: Bearer <access_token>`（D-16），permission 边界 = device pairing。

**LOC**：~250 LOC（server handler + App fetcher + AEAD），v0.2 实现。

**Server 端 v0.1**：不缓存，每次 GET 重新读文件 + SHA256 校验（防文件被改 → 返回 410 Gone）。v0.2.x 高负载再加 LRU。

**Blob path 安全规则（D4 决议，v0.1 实现 daemon 端 register endpoint stub 时遵守）**：

| # | 规则 | 实现 |
|---|---|---|
| 1 | path 规范化 | `filepath.Abs` + `filepath.Clean` |
| 2 | 拒绝路径穿越 | `filepath.Rel(allowed, path)` 不能以 `..` 开头 |
| 3 | symlink resolve + 冻结 | `filepath.EvalSymlinks`，resolve 后**再**做 Rule 2 检查；存 resolved path |
| 4 | allowlist 机制 | workspaceRoot 隐式 always allowed；`~/.tether/config.toml` 的 `blob.allowlist.extra_paths` 加额外路径（典型 `~/.cache/<skill-id>/`） |
| 5 | 文件必须存在 + 可读 | 注册时打开文件 SHA256；打不开拒绝 |

**Registry 持久化**：`<workspaceRoot>/.tether/blobs/registry.json`，atomic file write（tmp + rename）防 race。daemon 重启从此文件恢复。

见 D-12 / D-18（§2）。

### 11.D Lock state machine 细节 ✅ **已锁定**

**架构基础**（PoC-1 简化）：原本担心要做"双层 lock"（writer lock + permission_mode lock）。F-10/F-11 实测后发现 cc 自带三层权限模型（见 §5.5），**daemon 不需要造 permission 锁**——只需要管"谁能写 PTY 字节"这一层。

**Lock 决策表**（最终）：

| 子问题 | 决议 |
|---|---|
| 自动释放窗口 | **60s**（无 input 60s 后 holder=null） |
| 终端 force-takeover 按键 | **`Ctrl+\ Ctrl+T`** |
| Mobile force-takeover | **UI 按钮 + 二次确认 dialog** |
| view-only attach | **v0.1 不做**（理由：单用户场景多设备旁观非核心需求；多 client 已收到广播 envelope 默认就能"看"；"看但不抢锁"靠 lock idle 60s 自然成立——不主动发字节就不抢；增加 lock 状态机 + UI readonly 标识复杂度不值。v0.2 真有需求再加） |

**Permission_mode 切换**（不是 lock，是 RPC）：

Mobile App 通过 RPC `session-set-mode` 给 daemon 发模式切换请求。daemon 给 cc 发对应的 slash 命令或重启 cc with 新 `--permission-mode`。**这个权限是基于 lock holder 身份**：只有当前 holder 有权切 mode（避免抢锁的人偷换 mode）。

| RPC | 谁能调 | 副作用 |
|---|---|---|
| `session-set-mode { mode: "plan" }` | 当前 lock holder | cc 切到只读 |
| `session-set-mode { mode: "default" }` | 同上 | cc 回到默认（safe-cmd 跑、危险命令对话框）|
| `session-set-mode { mode: "auto" }` | 同上 | cc 自动放行（信任此会话）|
| `session-set-mode { mode: "bypassPermissions" }` | 同上 + UI 二次确认 | "yolo 模式" |

**Audit log**：每次 lock 状态变化（拿/释放/接管）+ mode 切换写 `~/.tether/users/default/sessions/<id>/lock.log`。

见 D-15（§2）。

### 11.E Push 通知架构 ✅ **已锁定（E2 + Pusher 多态抽象，APNs/FCM）**

**决定**：CLI 持 push 凭据 + 自己直接调推送服务 endpoint（**E2**），server 完全不参与 push。

**驱动**：
- C1（D-12）锁定后 server 是哑中继，不能看 push payload plaintext → server 触发 push 必须自己生成 payload 文本，跟 C1 冲突
- "CLI 离线时 server 还能推" 不是真需求——push 触发源（PreToolUse / Stop hook）都是 CLI 在线时产生
- server 少持一份秘密
- D-13 锁 Tauri Mobile 后，目标设备只有 iOS（APNs）和 Android（FCM），**无 Web Push**

**Pusher 抽象**（CLI 端，按 subscription type 多态）：

```go
type PushSubscription struct {
    Type     string          // "apns" | "fcm"
    Payload  json.RawMessage // type-specific（token 等）
    DeviceID string
}

type Pusher interface {
    Send(ctx, sub *PushSubscription, payload []byte, urgency string) error
}

pushers := map[string]Pusher{
    "fcm":  &FCMPusher{ServerKey},         // ★ v0.1 主路径（Android only by D-13）
    // v0.1.x 加："apns": &APNsPusher{Cert, BundleID}  ← iOS Mobile 启用时
    // 独立 milestone："jpush": &JPushPusher{...}      ← 大陆 Android 兜底
}
```

**配对时 Mobile App → CLI 发的 pushSub 字段**（通过加密 control envelope）：

```jsonc
// iOS (Tauri 通过 tauri-plugin-notification + APNs 拿到 token)
{ "type": "apns", "payload": { "token": "<APNs device token>" } }

// Android (Tauri 通过 tauri-plugin-notification + FCM 拿到 token)
{ "type": "fcm",  "payload": { "token": "<FCM registration token>" } }

// v0.2.x 加：大陆 Android (Tauri 集成极光 client SDK 拿到 registration ID)
// { "type": "jpush", "payload": { "registrationId": "<极光 RID>" } }
```

**🇨🇳 中国大陆 Android 兼容（独立 milestone，不在 v0.1）**：

FCM 在大陆**不可靠**——依赖 Google Play Services (GMS)，但国产 Android（小米/华为/OPPO/vivo/荣耀/魅族）出厂无 GMS，省电策略也阻 FCM 长连接。**v0.1 实操影响**：你跟第二用户的开发机要么 iOS、要么带 GMS 的 Android（自己装 GMS 测）。

**JPush 集成是独立 milestone，不是 60 LOC 小补丁**：
- CLI 端 REST API（~50 LOC）只是冰山尖
- **真成本**：App 端 JPush client SDK 集成（Android Java/Kotlin 桥到 Tauri Rust shell；隐私权限合规；厂商通道注册）+ 各国产厂商 manifest 配置 + 工信部隐私自检
- 估算 **5-10 person-days**，独立 milestone（v0.1.x 或 v0.2 才动）
- 备选：个推、友盟+ U-Push

**触发**：当真有大陆国产 Android 用户报"收不到 push"时，启动这个 milestone。在那之前 stick 在 FCM。

**CLI 触发 push** 的流程：

```
daemon 收到 PreToolUse hook → 决策"高优先级"
daemon → emit hook envelope (events stream) + 触发 push：
  pusher := pushers[device.pushSub.Type]
  pusher.Send(ctx, device.pushSub, payload, "high")

FCMPusher:  HTTP API + OAuth2 → POST 到 fcm.googleapis.com（v0.1 主路径）
APNsPusher: HTTP/2 + JWT 自签 → POST 到 api.push.apple.com（v0.1.x 启用 iOS 时加）
```

**Push 触发点**（F-06）：
- `PreToolUse`（危险命令需要审批）→ 高优先级 alert push："cc wants to run X"
- `Stop`（turn 完成）→ 中优先级 alert push："cc finished thinking"
- 其他 hook（SessionStart / UserPromptSubmit / PostToolUse）不推

**Push baseline 流程**（v0.1，所有平台默认）：

```
PreToolUse hook → daemon 决策推 → APNs/FCM alert push（含 hookEnvelopeId）
↓
用户看锁屏通知 → 主动点开 App → App 重连 WT → catch-up envelope → 显示审批 modal
↓
用户审批 → input.pre-tool-approval envelope → daemon → cc
```

**关键**：v0.1 **不依赖 silent push 后台 wake** —— 默认走"alert push 用户主动点开"路径。

**Silent push 增强（推迟到 v0.1.x，需 PoC 验证后启用）**：
- APNs `content-available: 1`（iOS） / FCM high-priority data message（Android）触发 silent wake
- Tauri shell 在 ~30s 窗口重连 WT 拉 hook envelope，弹 native 通知（用户能在锁屏审批，不必开 App）
- iOS 系统对 silent push 频率限流较严（每天 ~3 次保底）；FCM 高优先级也有平台限流
- **必须独立 PoC 实测可靠率 + 延迟**——不可靠时 v0.1 stick 在 alert baseline，silent 仅作锦上添花

**v0.1 Android-first 顺序**（D-13）：
1. **FCMPusher 先实现**（v0.1 主路径，Google Play 内测）—— ~50 LOC + OAuth2 service account
2. **APNsPusher 推迟到 v0.1.x**（iOS Mobile 启用时再加）—— ~80 LOC + JWT
3. JPushPusher 大陆 Android 兜底独立 milestone（见下方）

**v0.1 LOC 估算修订**：
- Pusher interface + 注册表：~30 LOC
- FCMPusher（HTTP API + OAuth2）：~50 LOC
- devices.json schema + pushSub 字段：~10 LOC
- **v0.1 合计 ~90 LOC**（无 APNs；APNs 留 v0.1.x ~80 LOC 增量）

见 D-12 / D-13。

### 11.F PTY 写入边界细节 ✅ **PoC-1 已答**

**已解决**：见 §5.3 + Appendix B 的 F-01 / F-07。
- Submit = `\r`（CR），不是 `\n`（LF）
- bracketed paste：cc 启动时主动启用 `\x1b[?2004h`，daemon 包 `\x1b[200~...\x1b[201~` 即可
- 取消 turn = `\x1b` + `\x15` + 新 prompt + `\r`
- slash 命令：直接 write `"/clear\r"`，cc 原生识别

**仍待 PoC**：multi-line 用户消息（含 `\n` 的 prompt）的 multi-line marker 形式——cc 内部到底用什么 escape，cc-cli 文档没明示，需要补一个微 PoC。

### 11.G App 壳子范围 ✅ **已锁定（Tauri 2 Mobile + Desktop，v0.1 Android + 桌面三平台）**

**决定**：v0.1 ship **Android（Tauri 2 Mobile）+ Desktop（Tauri 2 Desktop，覆盖 macOS / Linux / Windows）**，详见 D-13 / D-19 / §11.V / §11.Y。**iOS 推迟到 v0.1.x**。**不出 PWA 浏览器版**。UI 是 Web 技术（React/Svelte + Vite），跑在 Tauri WebView，桌面 + 手机共用一份代码（layout container 不同，详见 §11.Y）。

**为什么 v0.1 不含 iOS**（D-13 review 后调整）：
- iOS 路径有 3 个未验证的不确定性：iOS WT 真机 smoke test / APNs silent push / Apple Developer 流程 + 审核延迟（首次 1-2 周）
- Android 路径相对干净：FCM 标准、Google Play 内测渠道（小时级别 vs Apple 审核以周计）
- 单人 v0.1 时间线（14-16 周）下"全平台并行"是高风险——iOS 一旦在实现期出问题，整个 milestone 都被拖
- v0.1.x 加 iOS 增量改动主要在 APNs + 签名审核 + iOS 真机 smoke test——transport 层（tether 自写 Tauri command 包 `web-transport-quinn`）已是统一实现，**不动**，所以延后成本可控

**为什么不走 PWA 过渡**：UI 代码 PWA = Tauri（同 React/Svelte + Vite 跑 WebView），先 PWA 没省 UI 工程量；唯一节省的"加 Tauri 壳" 1-2 周相比 D-13 决议节省的 iOS 端复杂度收益更小。

**v0.1 Tauri Mobile (Android) 栈**：
- WebView：Chromium 原生（但 WT 不走原生 API；走 Tauri command 见下）
- WT：**tether 自写 Tauri command** 包 `web-transport-quinn`（与桌面三平台同一份代码；review 2026-04-28 P1 #5 = ii-rebuilt；详见 §11.V）
- Push：FCM via `tauri-plugin-notification`
- 长期密钥：Android Keystore (StrongBox) via `tauri-plugin-keyring`
- 持久化：Tauri FS plugin
- QR 配对：`tauri-plugin-barcode-scanner`

**v0.1 Tauri Desktop 栈**：详见 §11.V。同一份 WT Tauri command 在 macOS / Linux / Windows 三平台跑。

**v0.1.x iOS 增量**（独立 milestone）：复用同一份 tether 自写 WT Tauri command（已跨平台过 v0.1）+ APNs JWT pusher + Apple Developer 账号 + App Store 审核流程 + iOS 真机 smoke test。详见 §11.V。

**问题**：v0.1 Mobile App 实现哪些路由 + 哪些控件？

**路由选项**：
- `/` session 列表
- `/session/:id` chat 视图
- `/session/:id/file?path=` 文件查看
- `/session/:id/files` git 改动 sidebar
- `/pair` QR 配对（用 `tauri-plugin-barcode-scanner`）
- `/settings` 服务器 URL / push / 主题

**chat 视图必须有的控件**（§5.5 / §11.D 推论）：
- 消息流（user / assistant 文字 + tool_use / tool_result 折叠卡片）—— **数据来自 `output.agent-event`**（cc 写的 JSONL 文件，daemon fsnotify 增量推过来；不解析 PTY 字节，不需要 xterm.js）
- 输入框 + 发送 + "停止"按钮（停止 = 触发 §5.3 的 cancel-turn 序列）
- **Permission_mode 切换器**（plan / default / auto / bypassPermissions 四档；点击发 `session-set-mode` RPC）
- Lock holder 状态显示 + "接管"按钮
- Tool 审批弹窗（PreToolUse hook 触发；显示 tool_name + tool_input；按钮 allow / deny）

**渲染原则**：**原生 chat UI，不渲染 PTY 字节流**。Stream 3 agent-bytes v0.1 不在 WT 上消费（详见 §3.3.3 / §5.1）。这跟 happy 的做法对齐——chat UI 由结构化 jsonl-event 驱动。

**渲染 pipeline 抽象**（D-18 预留 v0.2 升级口子）：

assistant message 内容**不能**直接 `<ReactMarkdown>{content}</ReactMarkdown>`，必须走可扩展 pipeline：

```typescript
type ContentBlock =
  | { kind: 'prose'; text: string }
  | { kind: 'code'; lang: string; text: string }
  | { kind: 'fenced-extension'; lang: string; payload: any };  // v0.2 启用

function parseContent(text: string): ContentBlock[] { ... }    // v0.1: 仅 prose + code
function renderBlock(block: ContentBlock): ReactNode { ... }   // v0.1: prose → markdown, code → fenced；v0.2 加 dag/form/candidates/media

function renderMessage(content: string): ReactNode {
  return parseContent(content).map(renderBlock);
}
```

v0.2 Phase 1 把 streaming FSM（参考 v0.2 spec §4）接进 `parseContent`，加 `dag/form/candidates/media` 识别 + 对应 component renderer。**v0.1 留这个抽象层不做实际工作**，~50 LOC。

**程序触发输入接口三件套**（D-18 / D6 决议）：

App UI 暴露 **3 个独立 API**——分别对应三种 input envelope kind，**禁止合并**（语义不同，混用会出 bug）：

```typescript
// 1. 用户消息（v0.2 button-to-command 也走这个）
sendUserMessage(
  text: string,
  options?: {
    showInTimeline?: boolean,                       // 默认 true
    sourceHint?: "typed" | "button" | "shortcut",   // D7 attribution
    sourceContext?: any,                             // 可选：button 触发时记 {dagId, nodeId, action}
  }
): Promise<{ envelopeId: string }>;

// 2. 停止当前 turn（"停止"按钮走这个）
cancelCurrentTurn(): Promise<{ envelopeId: string }>;
// daemon 收到后触发 §5.3 的 Esc + Ctrl+U 序列，不是发字符串

// 3. 响应 hook（PreToolUse 审批走这个）
respondToHook(
  hookId: string,
  decision: "allow" | "deny" | "ask",
  options?: { reason?: string }
): Promise<{ envelopeId: string }>;
```

**内部共享 emitter**（不暴露给业务代码）：`emitInputEnvelope(kind, payload)` 负责加密 + envelope 包装 + WT stream 1 发送。

**v0.2 button-to-command 完整流程**（D7 决议）：

```
用户点 DAG 节点 f3 的 "Approve" 按钮 →
  sendUserMessage("/dag:approve abc f3", {
    showInTimeline: true,
    sourceHint: "button",
    sourceContext: { dagInstanceId: "abc", nodeId: "f3", action: "approve" }
  })
→ App 本地 timeline 加 user message bubble（**视觉跟手打无差**）
→ App 把 DAG f3 节点标"pending update"（subtle 视觉提示）
→ daemon 写 PTY → cc 处理 slash → 改状态 → emit 新 dag block
→ App 渲染更新，"pending update"清除
```

**Timeline 显示策略**（D7）：默认 button-triggered message **看起来跟用户打字完全一样**（Phase 1 spec §5.4 step 4 "as if the user had typed it"）。**长按 message bubble 弹 bottom sheet** 显示：
- 完整文本
- Source: `typed` / `button` / `shortcut` + sourceContext（"来自 DAG: storyboard-abc, 节点 f3"）
- 发送时间 + envelope ID

满足"clean log + audit on demand"两个目标。

**Pending update 超时**：v0.1 仅留 component state hook（`ComponentPending = { componentId, triggeredBy, startedAt }`），具体 timeout 阈值 + 超时 UX 由 v0.2 实现期定（建议 30s）。

**约束**：通勤场景核心是 chat + 审批 tool + 切 mode；文件查看相对低频。

**v0.1 范围**：必须有 `/`、`/session/:id`、`/pair`、`/settings` 路由 + 上面 5 个 chat 控件 + 渲染 pipeline 抽象 + `sendUserMessage` API；文件相关延到 v0.1.1。

### 11.H Server 存储 ✅ **已锁定（SQLite）**

**决定**：**SQLite**（embedded，单文件 `~/.tether/server/state.db`）

**驱动**：v0.1 单用户规模极小；SQLite 简单 + 持久 + 零运维 + 跟单二进制分发匹配。Postgres 留给 v0.2 多用户。

**Schema 草稿**（详细 schema 在 server 实现时定）：
- `devices` — 设备表（pubkey、type、pushSub、lastSeenAt）
- `sessions` — session 注册表（最低限度元数据，不存内容）
- `pending_envelopes` — B4 暂存队列（id / sessionId / kind / ciphertext / ts / TTL）
- `audit` — auth / pair / takeover / mode-switch 关键事件

### 11.I 部署形态决策 ✅ **已锁定（I3，I1 default）**

**决定**：**I3 双路径并行支持**，但 **I1（server + CLI 同机）是默认推荐路径**。

**v0.1 用户路径**：
- dogfood 期：server + CLI 同机（笔记本本地）→ 测 wire 协议、配对流程
- 稳定后：迁移到 GCP VM（仍是 server + CLI 同机）→ 通勤场景真用
- 未来如果有需求：I2（server 独立部署 fly.io/Railway，CLI 跑笔记本）也支持，但**不是 v0.1 推荐**

**约束**：自托管为主，v0.1 不做 hosted service。

### 11.J 配对 / Auth 复杂度 ✅ **已锁定（J2 + D-12 配对协议）**

**决定**：
- **配对协议**：D-12 / C1 已锁——X25519 ECDH + QR 直扫。所以"配对"层不需要再讨论
- **Auth token 层**：**J2** —— access token (短 TTL，~1h) + refresh token (长 TTL，~30 day)；无 token-family / grace-window / divergence detection（happy 那套多用户场景才需要）。盗用场景下 = `tether revoke <deviceId>` 重新配对

**单用户场景的简化理由**：
- token-family 主要解决"多设备并发刷新 + race condition"——v0.1 单用户少量设备没这个压力
- grace-window 主要解决"refresh token 失效但旧 access token 还在用"——简化到"refresh 失败立即重新配对"
- divergence detection 主要解决"被盗用但用户不知道"——v0.1 设备数少，每个 device 独立 pubkey 已经隔离，盗用单个 device 不影响其他

**LOC 估算**：
- 配对协议（已含 D-12 ~120 LOC）
- Token 颁发 + refresh + revoke：~150 LOC
- Server 端 token 验证中间件：~50 LOC
- **合计 Auth 层 ~200 LOC**（不含 D-12 配对）

见 D-16（§2）。

### 11.K Connectivity 文档 ✅ **已锁定**

**问题**：用户要怎么把 server 暴露出去给 Mobile App 连？

**决定**：v0.1 写 **3 套 setup guide**：
- **Tailscale Funnel**（推荐：跟 GCP VM 配合最顺、零配置 TLS）
- **Cloudflare Tunnel**（备选：完全免费、零信任 friendly）
- **公网 IP + Caddy / Let's Encrypt**（hardcore：用户自己控制全栈）

**fly.io 部署**作 v0.1.1 选项（跟 §11.I 的 I2 路径联动），免去自己暴露端口。

**所有 setup guide 必须覆盖**：
- QUIC 跑 UDP/443，需要确认网络放行（不少企业代理只放 TCP）
- **⚠️ 客户端 system proxy 必须 bypass server IP**（P-06，PoC-2 step 3 实测踩坑）：用户机器装 Clash / Surge / V2Ray 等 HTTP/SOCKS proxy 时，WT/QUIC 流量会走 proxy 被拒（HTTP CONNECT 不支持 UDP）。文档要明示给 server IP 加 DIRECT 规则。Clash Verge 示例：merge 文件加 `prepend-rules: [IP-CIDR,<server-ip>/32,DIRECT]`

### 11.L CLI 命令列表 ✅ **已锁定**

**决定**：单二进制 `tether`，多子命令分发。**`tether server` 也是同一个 binary 的子命令**（§11.I 选 I3 路径，不分 binary 简化分发；server 模式跟 daemon 模式靠 subcommand 区分）。

```
# 进程模式
tether daemon                   主进程（含 daemon goroutine + client goroutine），foreground
tether daemon --detach          后台模式
tether server                   独立 server 模式（I2 部署用，v0.1 同 binary）

# Session 操作（v0.1 仅 claude-code provider，无 --agent flag）
tether spawn [path]                  起 cc session（workspace = path）
tether spawn --workspace <path>      显式 workspace root（默认 = path 参数；v0.2 dag skill 用 <workspace>/.tether/）
tether spawn --mode <mode>           initial cc permission_mode（plan/default/auto/bypassPermissions）
tether attach <id>              raw-mode TTY 客户端
tether ls                       列 session（含 workspaceRoot）
tether resume <id>              重新拉起 cc（用 meta.json 里持久化的 mode + workspace）
tether mode <id> <new-mode>     切换 cc permission_mode（等价 Mobile App 的 session-set-mode RPC）
tether kill <id>                SIGTERM cc + 清理

# 配对 / 设备管理
tether pair                     生成 QR
tether devices                  列已配对设备
tether revoke <deviceId>        吊销 device（默认只吊销该 device）
tether revoke --rotate-all      高强度：吊销 + master rotation，所有 device 重配对

# 调试 / 诊断
tether status                   uptime / session 数 / 连接数 / 暂存队列大小 / decrypt failures
tether log <sid>                查单 session 的事件流（解密后人类可读）
tether doctor                   自检 meta/keys/devices/queue 一致性 + 修复建议
tether support-bundle           生成可分享 debug 包（redacted，无 plaintext）

# v0.2 Phase 1 预留（D-18，v0.1 stub）
tether-blob-register <path>     Skill 用：注册本地文件 → 输出 tether://blob/<sha256>
                                v0.1：返回 "not implemented" 错误，但命令存在让 skill 代码可引用
                                v0.2：实际调 daemon /blob/register
```

**Spawn 时副作用**（D-18 预留）：每次 `tether spawn` 自动 `mkdir -p <workspace>/.tether/`——v0.2 dag skill / blob registry 都在这个子目录下，提前创建让目录约定从 v0.1 起就稳定。

### 11.M 加密原语 ✅ **已锁定（随 §11.C C1 决议）**

**决定**：
- Key exchange：**X25519**（`golang.org/x/crypto/curve25519`）
- Symmetric AEAD：**XChaCha20-Poly1305**（`golang.org/x/crypto/chacha20poly1305`）
- Key derivation：**HKDF-SHA256**（`golang.org/x/crypto/hkdf`）

**理由**：
- 24B nonce 可全 random，不用持久 counter，dev 友好
- XChaCha20-Poly1305 在 2^32 消息后 nonce 冲突概率仍 2^-160
- AES-256-GCM 评估后驳回：12B nonce 必须持久 counter，daemon 重启边界处理麻烦

详见 §11.C。

### 11.N JSONL → Wire envelope mapper ✅ **已锁定**

**决定**：参考 happy 的 `sessionProtocolMapper.ts` 边界处理逻辑，**用 Go 重写一份精简版**（~300-400 LOC）。

**精简比 happy 少的原因**：
- happy mapper 同时处理 SDK 输出 → JSONL 的逆向逻辑（远程 SDK 模式）；我们 D-05 锁了纯 PTY 模式，**不需要 SDK 逆向**
- happy mapper 处理 sidechain / subagent / orphan buffering 边界——我们保留这些（cc 行为相同）
- PoC-1 F-04 实测 cc JSONL 三类：mapper **只处理 EVENT 类**（type=user/assistant，含 UUID 父子链）；HOOK 类走 hook server；STATE 类直接 session 视图更新

**核心责任**：
- 解析 JSONL 行 → 分类 (EVENT/HOOK/STATE，§5.6 / F-04)
- EVENT 类映射成 wire `output.agent-event` envelope（保留 cuid2 标识、UUID 父子链、sidechain 标记）
- 防御 torn-write（§5.6 / F-09）：末尾非 `\n` 视为未完整

**Transparent contract（D-18 / D5 决议）**：
- mapper **不解析 markdown**、**不识别 fenced block**（` ```dag `, ` ```form ` 等）
- cc JSONL 的 `assistant.message.content` 数组**原样**塞进 AgentEvent payload（保留 cc 的 text / tool_use / tool_result 块结构）
- text 块内的 markdown 字面值是 opaque——v0.2 fenced block 解析在 **App 端 streaming FSM**（参考 `2026-04-27-phase1-dag-protocol.md` §4）
- 这条 contract 让加新 fenced block 类型只改 App，不动 daemon / wire schema / provider 抽象

### 11.O 跨平台 ✅ **已锁定**

**决定**：
- **CLI**：v0.1 仅 **Linux + macOS**（amd64 + arm64）。creack/pty 原生支持。Windows v0.2 再看（WSL 推荐 / ConPTY 评估）
- **Server**：跨平台无障碍，但实际部署只在 Linux（GCP VM 等）
- **Mobile**：iOS + Android（Tauri Mobile，D-13）

### 11.P 分发 ✅ **已锁定**

**决定**：
- **CLI**：
  - **GitHub releases binary**（amd64 / arm64 × linux / macos × static linked）—— 主分发渠道
  - **Homebrew tap**：`brew install xiaokangw/tap/tether` —— macOS 用户首选
  - Docker image：v0.1.1 选项（容器用户）
  - 不发 `go install`、不发 npm
- **Mobile**：
  - **iOS**：App Store
  - **Android**：Google Play 内测渠道（v0.1）→ 公开（v0.1.x）
  - 不发 sideload APK（除非用户明确要 dev build）

### 11.Q 测试与 CI ✅ **已锁定**

**决定**（v0.1 测试矩阵）：

| 类型 | 内容 | 优先级 |
|---|---|---|
| **CC 集成测试** | PoC-1 的 12 步直接转 CI（覆盖 F-01 ~ F-11 全部事实，~1900 LOC Go 已写好；起真 cc + PTY 输入输出 + hook 通路 + JSONL 解析） | **P0** |
| Transport / wire | PoC-2/3 同款 转 CI（WT 双向 + connection migration + watchdog 注入 panic + envelope schema round-trip）| P0 |
| 单元测试 | crypto round-trip、lock 状态机、JSONL mapper、Pusher 实现 | P1 |
| Mobile E2E | `tauri-driver` + WebDriver 跑 chat 流程；Android emulator + iOS simulator | P1 |
| 跨平台 | CI matrix `linux × {amd64, arm64}` + `macos × {amd64, arm64}` | P0 |
| Stress | cc 大段输出 + Mobile 4G 模拟 + backpressure trim 触发；100 次随机崩溃 → resume 不丢消息 | P1 |

### 11.R 可观测性 ✅ **已锁定**

**决定**：
- 结构化 **JSON log** 到 stderr / file（按级别 INFO/WARN/ERROR；module 字段标识子系统）
- **`tether status`**：uptime / session 数 / 连接数 / 暂存队列大小 / 心跳延迟 / decrypt failure count / WT stream 状态 / push provider 最近响应
- **`tether log <sid>`** 查单 session 的事件流（解密后人类可读）
- **`tether doctor`**（P1-6 review 加）：自检 meta.json / keys.bin / devices.json / server pending queue 一致性；输出"健康 / 可 resume / 需重新配对 / 只能新建 session"四类状态 + 修复建议
- **`tether support-bundle`**（P1-10 review 加）：生成可分享 debug 包（redacted）。包含：
  - envelope lifecycle log（最近 N 条 id / kind / sessionId / direction / timestamp / size，**不含 plaintext**）
  - WT stream state（每 stream 收/发字节数、QUIC stream credit、reconnect reason history）
  - decrypt failure count + 最近 5 次失败的 envelope id + AD（不含密钥）
  - push provider 响应（FCM 最近 N 次的 status code、retry 次数）
  - lastSeenEventUuid（per session）+ server pending queue depth
  - JSON log 最近 N MB
- **Prometheus metrics** 推迟到 v0.2（自托管单用户不需要）

**Audit log schema**（P2-15 review 加，落盘 `~/.tether/users/default/audit.log`，JSON Lines）：

```jsonc
{
  "ts": "<ISO8601 with TZ>",
  "event": "pair" | "device-revoke" | "lock-takeover" | "lock-auto-release" |
           "resume" | "mode-switch" | "failed-auth" | "session-end" |
           "blob-register" | "ringbuf-overflow" | "trim-cc-bytes" | "catchup-needed",
  "sessionId": "<uuid or null>",
  "actorDeviceId": "<deviceId>",            // 哪个 device 触发
  "actorPubkeyFp": "<hex 8B>",              // device pubkey fingerprint
  "actorRemoteAddr": "<ip:port or 'local'>",// 仅 audit 用，不参与决策
  "tokenId": "<access token id or null>",   // 用于"哪个 token 干的"溯源
  "prevState": <event-specific>,            // e.g., 旧 lock holder, 旧 mode
  "newState": <event-specific>,             // e.g., 新 lock holder, 新 mode
  "outcome": "success" | "denied" | "error",
  "reason": "<text or null>"                // 失败 / takeover 原因
}
```

**约束**：自托管单用户场景，不上 Prometheus / OpenTelemetry 等基础设施。

**cc 兼容性 CI**（P1-13 review 加，§11.W `version.go` 配套）：
- 每个 cc 版本要 ship `tether` binary 前必须 PoC-1 12 步全过（`make compat-test CC_VERSION=<x>`）
- `internal/agent/claudecode/version.go` 写死 supported cc version range；启动时 `claude --version` 不在 range 即 hard-fail，提示用户降级 cc 或升级 tether
- CI matrix 覆盖 cc 当前 stable + 上一个 stable（应对 user 升级 cc 但 tether 没跟上）

### 11.X v0.2 Phase 1 forward-compat 预留 ✅ **已锁定（D-18）**

**v0.2 spec**：`2026-04-27-phase1-dag-protocol.md` —— tether 从"远程 cc 终端"升级到"远程 cc + 结构化 UI 客户端"（fenced block + button → slash + blob proxy）。

**这些预留是"路径占位 + 关键架构 seam"，不是"v0.1 已经把 DAG ready 好了"**。v0.2 真正的工作量（streaming FSM、4 类 component renderer、按钮 command injection 安全、blob auth/cache/lifecycle、状态重放协议）仍在 v0.2 实作阶段。9 项预留的价值是让那时候不需要破坏性升级 wire 协议或目录约定。

**v0.1 必做 9 项预留**（共 ~135 LOC，避免破坏性升级）：

| # | 改动 | 落点 | LOC |
|---|---|---|---|
| 1 | Session meta 加 `workspaceRoot` 字段（默认 = spawn path；`--workspace` 显式覆盖；**不做 auto-detect**——避免 submodule/worktree/monorepo 踩坑） | §6.1 | ~10 |
| 2 | daemon HTTP server 用 `/hooks/` 前缀；`/blob/*` 命名空间留 501 | §5.2 | ~5 |
| 3 | server HTTP/3 router 预留 `/blob/*` path | §3.1 / §11.T | ~10 |
| 4 | Blob 完整 wire 形态（256KB chunk + XChaCha20-Poly1305 + AD 三层防御）+ path 安全 5 规则 + allowlist config | §11.C | ~5 v0.1 stub；~250 v0.2 实现 |
| 5 | App `parseContent → renderBlock` 可扩展渲染 pipeline（**D-19 升级为 v0.1 1st-class**：dag/form/candidates/media 四类 block renderer 全实现，桌面 + 手机共用，详见 §11.Y） | §11.G / §11.Y | ~50 stub → ~400 v0.1 实现 |
| 6 | App 三件套 input API（`sendUserMessage` / `cancelCurrentTurn` / `respondToHook`）+ `sourceHint`/`sourceContext` 字段 | §11.G | ~30 |
| 7 | `tether spawn` 自动 `mkdir -p <workspace>/.tether/` | §11.L | ~5 |
| 8 | `tether-blob-register` CLI stub | §11.L | ~30 |
| 9 | catch-up envelope `catchup.state-snapshot` kind（**B-4 锁定 payload**：每 (sessionId, skillName, blockId) 最近 envelope；daemon LRU 缓存 ~20 per (session, skill)，全局 1000 blocks / 10MB；详见 §11.Z.9） | §3.3.2 / §11.Z.9 | ~80（daemon LRU + emit hook + catch-up channel replay）|

**为什么现在做**：1/2/3/7/9 是**符号 / 目录 / wire path 占位**——零或近零成本，但事后改是破坏性升级（用户要重配对/重启/改 plugin 代码）。4 是**加密 contract**——加密设计仓促容易留洞。5/6 是 **App 架构形态**——硬编码渲染后改成 pluggable 要重构核心。

**附 D2 / D5 / D6 / D7 决议要点**（在 §11 各对应章节展开，此处仅索引）：

- **D2** 多 session 共享 workspace：状态文件 race 由 skill 端 file lock 处理；App component map 按 (sessionId, instanceId) 隔离 —— v0.1 wire/daemon 不感知
- **D5** Fenced block 解析归属：daemon `jsonl_mapper` 保持 transparent，content 数组原样转发；解析在 App-side per-provider renderer
- **D6** Input API 分工：三个独立 API 不合并（cancel-turn 是控制序列不是发字符串），共享内部 `emitInputEnvelope` helper
- **D7** Timeline 显示：button-triggered 默认视觉跟手打一样，长按 bubble 弹 bottom sheet 暴露 source/context/envelope ID（clean log + audit on demand）

### 11.W AgentProvider internal seam ✅ **已锁定（D-17，v0.1 不公开多 provider）**

**决定**：`AgentProvider` 是 daemon 内部 Go interface，**v0.1 仅 `ClaudeCodeProvider` 一个实现**，不暴露给 CLI 用户、不承诺多 provider 公开能力。接口形态在第二个 provider 实际实作时再稳定。

#### 11.W.1 v0.1 实现：ClaudeCodeProvider

代码组织 `internal/agent/claudecode/`：

| 文件 | 责任 | LOC |
|---|---|---|
| `provider.go` | `AgentProvider` interface 实现入口 | ~100 |
| `spawn.go` | PTY spawn cc（port from happy-cli `claudeLocal.ts`） | ~150 |
| `hooks.go` | 生成 cc settings.json 含 hook URL（port from `generateHookSettings.ts`）| ~100 |
| `hookserver.go` | daemon 本地 HTTP 端接 cc hook 回调（port from `startHookServer.ts`）| ~150 |
| `path.go` | `/x/y/z` → `-x-y-z` 编码（F-03） | ~30 |
| `findsession.go` | cc session 查找 | ~50 |
| `checksession.go` | cc 版本/health 检查 | ~30 |
| `specialcmd.go` | slash 命令处理（port from `specialCommands.ts`）| ~100 |
| `jsonl_watcher.go` | fsnotify + 增量解析 + 三类分类（F-04 / F-09）| ~150 |
| `jsonl_mapper.go` | JSONL → AgentEvent normalized 转换（§11.N）| ~300-400 |
| `pty_io.go` | PTY 输入清洗（F-01 / F-07）+ ring buffer fan-out | ~150 |
| `version.go` | cc 版本兼容 hard-fail（详见 §11.Q cc 兼容 CI） | ~50 |
| **合计** | | **~1330-1430 LOC** |

`SupportedModes()` 返回 `["plan", "default", "auto", "bypassPermissions"]`；`HookCapabilities()` 返回 cc 的 5 类 hook：PreToolUse / PostToolUse / SessionStart / UserPromptSubmit / Stop。

#### 11.W.2 Wire envelope `providerType` 字段

v0.1 envelope `output.agent-event` plaintext 含 `providerType` 字段，**v0.1 恒等 `"claude-code"`**——保留字段为 v0.2/未来留路，但 v0.1 不承诺多 provider 能力（无 `--agent` flag、无 `tether agents` CLI、App 端只一个 renderer）。

#### 11.W.3 第二 provider 加入时（不在 v0.1 范围）

如果未来真要加 codex / opencode / harness：
1. 那时**重新评估接口形态**——v0.1 单一实现可能因为只看 cc 一种行为有偏差
2. 接口 breaking change 走"加 capability flag"或重命名，不暴力砍方法
3. App 端加 per-provider renderer 走 `providerType` 字段分发
4. 决定是否暴露 `--agent` flag / `tether agents` CLI 给用户

**v0.1 不预先设计这些**——避免在没真实 case 时定错接口。

### 11.T CLI↔server transport ✅ **已锁定（CLI 也走 WebTransport-over-HTTP/3）**

**决定**：CLI 端用 `quic-go` 的 HTTP/3 client + `webtransport-go` 的 client mode 跟 server 建立 WT session，跟 Mobile App 完全对称。

**理由**：
- server 单一 listener 处理两边，envelope wire schema、stream 编排、datagram 语义全对称
- 调试时 CLI 和 Mobile App 抓出来的 wire 长得一样，问题排查复用
- quic-go + webtransport-go 库支持成熟（2026-01）
- v0.1 单用户场景下 raw QUIC 的"自由 framing 性能"完全不是瓶颈（envelope 流量 KB/s 级）

**Stream / datagram 编排**（Mobile App 和 CLI 共用，5 通道）：

详见 §3.3.3。简表：

| Stream | 用途 | 优先级 | 丢弃 |
|---|---|---|---|
| Stream 1 (control) | session.* / control.* / input.* | HIGH | 永不丢 |
| Stream 2 (events) | output.agent-event / output.hook-event | HIGH | 永不丢 |
| Stream 3 (agent-bytes) | output.agent-bytes（chunk 化）| LOW | backpressure 下 trim |
| Stream 4 (catch-up) | catchup.* | MEDIUM | 不丢 |
| Datagram | health.* | — | 天然 unreliable |

**为什么必须把 events 和 agent-bytes 拆成两个 stream**：QUIC stream 内严格顺序，**同一 stream 内 agent-bytes 大段 burst 会堵住后面的 hook-event 审批**。HTTP/3 PRIORITY 只在 stream 之间起作用，stream 内不行。

**子决策**：
- WT 端口跟 server HTTPS 同 **443**，靠 ALPN 区分
- Control stream **全局一条 + 内部 session-id 路由**，多 session 切换更便宜
- 关键事件（lock-takeover / permission-mode change）走 stream 不走 datagram；datagram 只承载 lossy-tolerable 信息
- **HTTP/3 path 命名空间预留**（D-18）：`/wt/*` for WebTransport sessions；**`/blob/*` 保留给 v0.2 Phase 1 blob proxy**（v0.1 不实现，但 router 表里别让其他 handler 占）

**LOC 估算**（transport 层）：
- server WT listener + auth handshake + stream router (5 通道) + datagram router + connection registry：~370 LOC
- CLI WT client + stream lifecycle + 重连（connection migration 帮大忙）+ datagram：~290 LOC
- 合计 **~660 LOC**

更详细的 wire 层 LOC（envelope schema / chunker / reassembler / trim / catch-up）见 §3.3.6。

**风险点 → PoC-2 必验**：webtransport-go 的 client mode 在长连接 + 心跳 + connection migration 下的稳定性。如果发现客户端 API 太薄，再降到 raw QUIC（破坏对称性，但不损失功能）。

见 D-02（§2）。

### 11.U 远程 attach 入口 ✅ **已锁定（U1）**

**决定**：v0.1 走 **U1（仅 Unix socket）**。用户笔记本不在 daemon 机器上时，先 SSH 到那台机器，再跑 `tether attach <id>`。

**理由**：
- 通勤场景下 99% 时间用户在 daemon 那台机器（GCP VM 或自己机器），SSH 已经是肌肉记忆
- 远程 attach 模式实质是把 `tether attach` 变成 Mobile App 的"终端版"——但 Mobile App 已经能完整覆盖通勤场景（Chat + 审批 + Mode 切换），多一个"终端 UI 走 server"路径价值低
- 实现远程模式需要重做 raw-mode 客户端走 QUIC（不再走 Unix socket）+ 走完整 envelope 加密链路 → ~300 LOC + 维护成本

**未来扩展（v0.1.x 选项）**：
- U2：`tether attach --remote <id>` —— 通过 server 中转，路径同 Mobile App
- 如果有用户报需求（"我经常在没有 SSH 的笔记本上"）再加

### 11.S 实现时间线

**问题**：v0.1 MVP 大概几周？

**预估**（按子模块，**v0.1 Android + Desktop + 全 surface**，review 2026-04-28 P1 #7 = X 重估后）：
- ~~PoC-1（cc 集成层）：1 周~~ → ✅ **已完成 1 天**（2026-04-27）
- ~~PoC-2 WebTransport HARD GATE：1 周~~ → ✅ **5/6 步通过 ~半天**（2026-04-28，§11.T 实质性通过）
- **PoC-2.6 `web-transport-quinn` × Android smoke test：1-2 天**（前置 Tauri build matrix，详见 §10）
- PoC-3 watchdog：~0.5 周
- daemon goroutine（cc + PTY + JSONL + hook）：2 周
- client goroutine + server 初版（QUIC + envelope 路由 + 配对）：2 周
- **App UI（全 surface：桌面三栏 + 手机单栏 + 4 类 fenced block renderer + workspace 树 + chat + pair + settings + error/a11y/i18n + catch-up replay）：5 周**（review P1 #7 重估 ~3000 LOC vs 原 780；详见 §11.Y.3）
- **Tauri Mobile (Android) + Desktop (macOS/Linux/Windows) 壳 + 自写 WT Tauri command + 分发**：~2.5 周（+1 周 vs 原 1.5 周覆盖 self-built WT polyfill ~500-1000 LOC，详见 §11.V / §10 PoC-2.6）
- **Skill 系统（D-20）：~0.5 周**（~470 LOC pure symlink，daemon 不感知 skill）
- E2E（C1 ~390 LOC）+ push (FCM only ~90 LOC)：1.5 周
- 测试 + 文档 + 自用 dogfood：2 周
- buffer：1-2 周

**v0.1 合计**：**14-16 周**（vs review 后 13-15 周 + 1 周自写 WT polyfill；review P1 #5 = ii-rebuilt 改为自写后净 +1 周，因为放弃了"用现成 plugin"的 0 LOC 假设）

**为什么 App UI 5 周而非 3 周**：原 780 LOC 估算遗漏 ~10 个关键模块（streaming FSM / markdown / state mgmt / routing / pair flow / settings / catch-up / error / i18n / a11y）。重估 3000 LOC 含全部，按 ~250 LOC/天 + Tauri 学习曲线 + UI 调试 + 设计迭代 ≈ 5 周。**X 锁定不 descope** —— 完整 v0.1 体验（含 form / candidates / a11y / i18n）兑现 D-19 「AI-generalized VS Code」承诺。

**为什么 Tauri 阶段 2.5 周而非 1.5 周**：自写 WT Tauri command 包 `web-transport-quinn`（~500-1000 LOC Rust + JS shim）+ 4 个 target build matrix 调通。原假设有现成 plugin 是 0 LOC，研究确认 plugin 废弃（review P1 #5 = ii-rebuilt 决议）后，本来要付的代价显式化。

**桌面 target 边际成本**：D-13/D-19 锁定后桌面与移动共用 Tauri 项目 + 共用 fenced block 渲染器；layout 切换 + workspace 树是增量。

**Skill 系统 0.5 周**：D-20 锁定 skill 全部 delegate 给 cc plugin loader（pure symlink）+ P0 #4 c 决议无 namespace rewrite，daemon 零运行时。

**并行性说明**：上面单子是顺序累加 ~16-17 周；实际可并行（如 E2E 加密 + push 在 daemon 实施中后期介入；测试 + dogfood 滚动进行）。**14-16 周** 已含合理并行假设。

**v0.1.x iOS 增量**（独立 milestone）：~1.5-2 周（含 Apple 审核不可控延迟）

### 11.V Tauri 平台栈（Mobile + Desktop） ✅ **已锁定（v0.1 Android + Desktop；iOS 推迟到 v0.1.x）**

**Tauri 2 状态**（2026-04）：2.10.3 已 1.5+ 年生产化；HMR 移动端可用；iOS 后台白屏 bug 在 2.10.x 已修；iOS App Store 案例 flow-like.com（2026-04 出货）；Desktop（macOS/Linux/Windows）三平台都已成熟生产化数年。

**单一 Tauri 项目，多 target build matrix**：D-13 / D-19 锁定后 v0.1 ship 三个 build target——Android (.apk / .aab)、Desktop macOS (.dmg / .app)、Desktop Linux (.deb / .AppImage)、Desktop Windows (.msi / .exe)。WebView UI 代码 100% 共享，layout 在运行时按 viewport 切换桌面三栏 / 手机单栏（详见 §11.Y）。

**v0.1 Desktop 平台栈**：

| | macOS | Linux | Windows |
|---|---|---|---|
| WebView | WebKit (WKWebView) | WebKitGTK | WebView2 (Chromium-based) |
| WT 实现 | **tether 自写 Tauri command** 包 `web-transport-quinn`（kixelated 维护，活跃 lib） | 同左 | 同左 |
| 长期密钥 | macOS Keychain | libsecret | Windows Credential Manager |
| 后台 | 桌面进程长存，无后台限制；不需要 wake/push | 同左 | 同左 |
| 分发 | DMG（自签或 Apple Notarization）| `.deb` + `.AppImage` | MSI signed |

**统一 homebrew polyfill 跨 Android + 桌面三平台**（review 2026-04-28 P1 #5 = ii-rebuilt）：所有 target 都走 tether 自写的 Tauri command（4-6 个 minimal command 包 `web-transport-quinn`），**不依赖 WebView 原生 WT，也不用 abandoned `tauri-plugin-web-transport`**。

**为什么 ii-rebuilt（自写）而非 ii（用 plugin）**：
- `tauri-plugin-web-transport`（kixelated）确认废弃：4 commits、"Maybe works fully?" 最后 commit、绑死 Tauri 2.5（我们要 2.10）、零 Android 测试
- Fork-and-fix 跟绿地写新代码同成本（plugin 本身就 4 commit 没东西可救）
- `web-transport-quinn` 是真正活跃维护的底层 Rust lib（234 stars / 281 commits / 活到 2026-04-21 / iroh + MoQ-rs 在生产用）
- **XY 简化**：tether 不需要 W3C `WebTransport` JS IDL 完整实现——我们自家 transport 调用自家 API 即可；暴露 4-6 个 minimal command（connect / openBidi / openUni / send / recv / close）+ 薄 JS shim，比抄 IDL 简单 ~1-2 天

**LOC**：~500-1000（Rust 4-6 个 Tauri command + JS 端 shim 模拟 enough WT 语义）。

**前置依赖 PoC-2.6 Android smoke test**：v0.1 commit Tauri build matrix 之前必须验 `web-transport-quinn` 在 `aarch64-linux-android` target 编译 + 真机 / emulator 跑通基本 WT dial（用户已有 Android 设备 + Android Studio emulator 都能覆盖）。详见 §10。**桌面 macOS / Linux / Windows** 编译跑通 quinn 是常规事，不算专门 spike——开发期一边写一边验。

**v0.1 Android 平台栈**（实际实现）：

| | Android v0.1 |
|---|---|
| WebView | Chromium 原生 |
| WT 实现 | **tether 自写 Tauri command** 包 `web-transport-quinn`（与桌面 + 未来 iOS 同一份代码；review 2026-04-28 P1 #5 = ii-rebuilt）|
| Push | FCM（OAuth2 service account） |
| 长期密钥 | Android Keystore（StrongBox） |
| 后台连接存活 | 较宽松（vs iOS） |
| 后台 wake | FCM 高优先级 data message + WorkManager |
| 分发 | Google Play 内测渠道（小时级别审核）|

**iOS 增量栈**（v0.1.x 启用，独立 milestone）：

| | iOS v0.1.x |
|---|---|
| WebView | WKWebView（不支持原生 WT）|
| WT 实现 | **tether 自写 Tauri command** 包 `web-transport-quinn`（与 v0.1 Android + 桌面三平台同一份）|
| Push | APNs HTTP/2 + JWT |
| 长期密钥 | Keychain（Secure Enclave）|
| 后台连接存活 | ~3min 系统断 networking |
| 后台 wake | alert push baseline；silent push 待 PoC 验证可靠率 |
| 分发 | App Store（首次审核 1-2 周） |
| 增量 LOC | ~80 LOC APNs JWT + Tauri WT plugin 集成 + Apple Developer 流程 |

**意外收益**：iOS 端 Rust quinn polyfill = CLI 端 quic-go 的对等 Rust 实现 → 未来 wire 调试可共用 quinn-tools dump，CLI 也可降级到同一份 quinn 代码（§11.T 风险点的天然 fallback）。

**v0.1 引入的 Tauri plugin**（Android only）：

| Plugin | 用途 |
|---|---|
| `tauri-plugin-notification` | FCM push 注册 + 显示 |
| `tauri-plugin-keyring` | 长期密钥落 Android Keystore |
| `tauri-plugin-fs` | session_key.bin / devices.json / queue 落盘 |
| `tauri-plugin-barcode-scanner` | QR 配对扫码 |
| `tauri-plugin-deep-link` | "打开 push 通知 → 跳转到 session" |

**v0.1.x iOS 加**：复用 v0.1 已写好的 tether 自写 WT Tauri command（同一份代码，不需新加 plugin）+ APNs 处理（在 `tauri-plugin-notification` 同插件下）

**前端代码 Tauri 适配纪律**：
- Native-ish API 经过 `@/platform/*` 抽象层（storage / notifications / share / file）
- 业务逻辑不直接调 Tauri API
- WebTransport client 包在 `@/transport` —— **单一 implementation**：调用 tether 自写 Tauri command（Rust 端包 `web-transport-quinn`）；四平台（Android + macOS / Linux / Windows，未来 iOS）行为 100% 对称
- Build 输出纯 SPA（Vite SSG）

**Hook 审批可靠性 baseline**（v0.1 不依赖 silent push）：
- **默认**：alert push 用户主动点开 App → 拉 hook envelope → 审批 modal
- **进阶**（v0.1.x 启用 iOS 后再 PoC）：silent push wake → ~30s 窗口拉 envelope → 锁屏 native 通知；iOS APNs 限频每天 ~3 次保底；Android FCM 高优先级 data 也有平台限流

**v0.1 实现成本**（Android + Desktop，含自写 WT Tauri command；全部位于 polyforge `tether-app` repo）：
- Tauri 2 Mobile + Desktop 项目脚手架（同一 `tether-app` repo 多 target）+ Android / macOS / Linux / Windows build：~3 天
- 各 Tauri plugin 集成（FCM / Keystore / FS / barcode / deep-link）+ `@/platform/*` 接线（mobile + desktop 适配分支）：~3 天
- **自写 WT Tauri command**（4-6 个 minimal command 包 `web-transport-quinn` + JS shim ~500-1000 LOC，跨四平台跑；落点 `tether-app/src-tauri/src/wt/`）：~5 天
- Android 签名 + FCM service account 配置 + Google Play 内测发布：~2 天
- Desktop 三平台打包 + macOS notarization + Windows code signing + 自托管 release / 简单 release page：~2 天
- **合计 ~2.5 周**（vs 原 Android only ~1 周，+1.5 周覆盖自写 WT polyfill + 桌面三平台；与 §11.S timeline 一致）

**v0.1.x iOS 增量成本**（独立 milestone，复用 v0.1 自写 WT Tauri command）：
- iOS target build + 真机 smoke test（同一份 WT 代码跨编译到 `aarch64-apple-ios`）：~3 天
- APNs cert + JWT signing + Apple Developer 账号申请：~3 天
- 首次 App Store 审核：1-2 周（不可控）
- 总：**~1.5-2 周**（取决于审核延迟）

**参考**：[Tauri](https://tauri.app/) | [`web-transport-quinn` (kixelated)](https://github.com/kixelated/web-transport) | flow-like.com（同期生产案例）

**已评估并放弃**：[`tauri-plugin-web-transport` (kixelated)](https://github.com/kixelated/tauri-plugin-web-transport) —— 4-commit hobby 项目，"Maybe works fully?" 最后 commit，绑死 Tauri 2.5（我们要 2.10），零 Android 测试。详见 D-13 / review 2026-04-28 P1 #5 决议。

### 11.Y App surface model（桌面三栏 + 移动 chat-first） ✅ **已锁定（D-19）**

**定位**：tether = **AI-generalized VS Code**。VS Code 把 IDE shell 通用化承载任意编程语言；tether 把 IDE shell 通用化承载**任意 AI 任务**。skill = "extension"——决定中栏内容 + chat 可用 verb。

#### 11.Y.1 桌面（主形态）

三栏布局，跟 VS Code 直接同构：

```
┌──────────┬─────────────────────────┬──────────────────┐
│          │                         │                  │
│ workspace│  skill 产物 + actions    │  AI chat (右栏)   │
│  树      │  （中栏，当前 active     │                  │
│ （左栏） │   skill 决定）           │  • 用户输入       │
│          │                         │  • agent 输出     │
│ ▾ xiyou/ │  ┌────────────────────┐ │  • inline cards  │
│   ep01/  │  │ DAG: xiyou-ep03    │ │  • skill verb 按 │
│   ep02/  │  │ N1 → N2 → N3       │ │    钮（rollback  │
│   ep03/  │  │   （current）      │ │    / approve / …）│
│   assets/│  │ [回滚] [详情]       │ │                  │
│ ▸ ieops- │  └────────────────────┘ │ ────────────     │
│   ws/    │  ┌────────────────────┐ │  (input box)     │
│          │  │ media: frame_001   │ │                  │
│          │  │ [缩略图] [赞][修]   │ │                  │
│          │  └────────────────────┘ │                  │
└──────────┴─────────────────────────┴──────────────────┘
```

**左栏（workspace 树）**：来源是 unified workspace root（D-19 + Q6：用户配置文件指定 root，全部 workspace 物理位于 root 下），渲染 = 真实目录树 + 当前 active workspace 高亮。点击切换 active workspace。

**中栏（skill 产物 + actions）**：当前 active skill emit 的 fenced block 全集渲染——dag node 图、media 全屏预览、form 输入、candidates 选择列表。每个 block 带 skill 定义的 action 按钮。中栏可多 block 滚动堆叠（横向并排单个 block 详情，或者纵向时间线列表，看 skill 偏好）。

**右栏（AI chat）**：跟 agent 的对话流，**复用 fenced block 渲染器**——但因为右栏宽度不及中栏，inline card 默认收起更紧凑（点开滚到中栏全屏视图，不在右栏自己撑开）。chat 输入框支持普通文字 + skill verb 命令（点 button → emit slash 到 agent）。

#### 11.Y.2 移动（commute-mode 压缩）

单栏 chat-first 流，桌面三栏的「commute 压缩」：

```
┌──────────────────────────────┐
│ [☰ workspace 抽屉]   xiyou ▼ │ ← 顶栏：workspace 抽屉（左栏压缩）+ 当前 ws
├──────────────────────────────┤
│  我: 继续 ep03                │
│                               │
│  agent: 已 dispatch N3        │
│                               │
│  ╔═════════════════════════╗ │ ← 中栏 dag block 压缩成 inline card
│  ║ DAG: xiyou-ep03         ║ │
│  ║ N1→N2→N3(current)       ║ │
│  ║ [全屏] [回滚]             ║ │   点 [全屏] = 桌面中栏视图
│  ╚═════════════════════════╝ │
│                               │
│  ╔═════════════════════════╗ │ ← media block 同样压缩成 card
│  ║ ep03_frame_001.png      ║ │
│  ║ [缩略图]                  ║ │
│  ║ [全屏] [赞] [修]          ║ │
│  ╚═════════════════════════╝ │
│                               │
│  ─────────────────────────── │
│  [输入框]  [发送]              │
└──────────────────────────────┘
```

**抽屉左滑**：workspace 切换；**点 card [全屏]**：进入跟桌面中栏同一份「单 block 详情视图」组件；**输入框**：跟桌面右栏共享同一份输入组件。

**核心承诺**：手机端**不丢功能**——只是 layout 压缩；任何桌面能做的操作（回滚 dag 到任意 node、填 form、选 candidates、看 media 详情、给 media 加 comment）手机都能做，路径多 1-2 次点击但语义对等。

#### 11.Y.3 共享渲染器架构

```
                    ┌─────────────────────────┐
                    │ fenced block 4 类         │
                    │ renderer 组件库          │
                    │   <DagBlock />          │
                    │   <FormBlock />         │
                    │   <CandidatesBlock />   │
                    │   <MediaBlock />        │
                    └────────────┬────────────┘
                                 │
              ┌──────────────────┼─────────────────┐
              │                  │                 │
              ▼                  ▼                 ▼
       桌面右栏 chat       桌面中栏全屏      移动 chat 流
       （inline 紧凑）    （单 block 撑满）  （inline card）
       cardLayout=compact cardLayout=full  cardLayout=compact
```

每个 block 组件接受 `layout: "compact" | "full"` prop，自己决定密度——同一份代码三个 layout 容器复用。

**LOC 估算**（v0.1 实现，review 2026-04-28 P1 #7 重估 = X 全 surface 保留，无 descope）：

原 780 LOC 估算遗漏 streaming FSM / markdown 集成 / state management / routing / pair flow / settings page / catch-up replay / error boundaries / a11y / i18n 等关键模块。重估完整列表：

| 模块 | LOC | 说明 |
|---|---|---|
| `<DagBlock />` | ~180 | 节点图 + 拖拽布局 + 回滚 action + a11y basic |
| `<FormBlock />` | ~200 | 输入字段 + 客户端 validation + 提交 + pending 状态 + 错误显示 |
| `<CandidatesBlock />` | ~140 | 多选列表 + 选中高亮 + 提交 + 取消 |
| `<MediaBlock />` | ~220 | 图/视频/音频 preview + 全屏 + 缩略图 + actions + 加载/错误态 |
| `<WorkspaceTree />` | ~180 | 树渲染 + 展开/收起 + active 高亮 + 50 ws 时基础 virtualization |
| `<DesktopLayout />` | ~160 | 三栏 + 拖拽分隔 + viewport 切换到手机布局 |
| `<MobileLayout />` | ~140 | chat 流 + workspace 抽屉 + 全屏 push + filter chip |
| Chat input + slash command emit | ~180 | 输入 + 历史 + 命令补全 stub + send button |
| **Markdown 集成 + theming** | ~150 | remark/marked 集成 + dark/light theme + 代码块基础高亮 |
| **State management + routing** | ~250 | Tauri webview 路由 + persistent UI state（zustand 或类似）+ 路由守卫 |
| **Catch-up replay handling** | ~120 | envelope LRU 接入 + 渲染顺序还原 + skill filter chip 状态 |
| **Pair flow UI**（review P1 #11）| ~350 | desktop 发起方 + mobile companion 共 6-8 屏（QR 显示 / 扫码 / SAS 输入 / fingerprint match / 成功）|
| **Settings page** | ~180 | 账号 / skill 列表 + 启用切换 / connection / about 4 区 |
| **Error boundaries + toast** | ~200 | 顶层 error boundary + toast 系统 + 重试逻辑 + 网络错误专属处理 |
| **i18n（en + 中文）** | ~150 | i18n 框架接入 + 全文本 key 抽取 + 2 locale 翻译 |
| **a11y（基础合规）** | ~200 | keyboard nav + focus management + ARIA roles + screen reader labels |
| **合计 v0.1 App UI** | **~3000 LOC** | (vs 原 780 LOC 估算，~3.8x 上调) |

**为什么按 X = 全 surface 保留**（review 提了 Y 系列 descope + Z 分两段，brainstorm 锁定 X）：
- 桌面三栏 + 4 类 renderer + a11y / i18n / pair / settings / error 全部进 v0.1
- 不 descope 任何 surface
- App UI timeline 由 3 周 → 5 周（详见 §11.S）
- 总 timeline 11.5-13.5 周 → 13-15 周（P1 #7） → **14-16 周**（P1 #5 = ii-rebuilt 自写 WT polyfill +1 周；详见 §11.S）

按 ~250 LOC/天 + Tauri 学习曲线 + UI 调试 + 设计迭代，~5 周可塞下 3000 LOC。

#### 11.Y.4 skill 作者承诺

skill emit fenced block stdout（在 cc 输出 / hook callback / agent provider event 任意一处），**不关心两端 layout**——tether shell 负责 desktop / mobile 适配。这跟 VS Code extension 不画 panel chrome 是同一个原则。

skill 可以在 fenced block 里指定 hint：
```yaml
preferred_layout: full       # 偏好桌面中栏全屏（用户默认进 detail view）
mobile_default: collapsed    # 手机默认收起，节省 chat 空间
actions: [rollback, edit, comment]   # 该 block 支持的 verb 集合
```

这些是 hint，shell 可覆盖；但 skill 作者据此可控制大致体验。

#### 11.Y.5 Desktop / Mobile transport ✅ **已锁定（D-8=A，desktop 永远 WT-via-server，与 mobile 同路径）**

**前提**（与 §11.I I1 部署形态一致）：daemon 与 server **同机部署**在用户的 GCP VM / 工作站上；desktop 与 mobile 都是 daemon 的**远程**客户端，**永远不与 daemon 同机**。Desktop 跑在用户笔记本/平板，daemon 跑在服务器侧。

**结果**：desktop 和 mobile 用**完全相同**的 transport 路径——WebTransport-over-HTTP/3 → server → daemon。无 UDS 直连分支，无 transport probe 逻辑。

```
┌─────────────────────┐         ┌─────────────────────┐
│ Desktop (Tauri)     │         │ Mobile (Tauri)      │
│ macOS/Linux/Windows │         │ Android (v0.1)      │
└──────────┬──────────┘         └──────────┬──────────┘
           │                               │
           │ WebTransport-over-HTTP/3      │ WebTransport-over-HTTP/3
           │ (与 mobile 完全对称)           │
           └───────────────┬───────────────┘
                           ▼
                ┌─────────────────────┐
                │ tether-server       │ ← 同机
                │ (HTTP/3 listener)   │
                └──────────┬──────────┘
                           ▼
                ┌─────────────────────┐
                │ tether daemon       │ ← I1 部署
                │ + cc                │
                └─────────────────────┘
                  GCP VM / 工作站
```

**单一 transport 代码路径**：
- `@/transport` 抽象层只有一个 implementation：WT client via **tether 自写 Tauri command**（Rust 端包 `web-transport-quinn` polyfill，跨 Android + desktop 三平台一致；review 2026-04-28 P1 #5 = ii-rebuilt；详见 §11.V）
- 不需要 UDS adapter / probe / fallback 逻辑
- desktop 跟 mobile envelope wire 100% 对称——调试体验对等
- transport implementation 也对称（不分 native vs polyfill），调试 / 测试矩阵减半，Chromium WebView 升级周期解耦

**`tether attach` 不变**：仍走 daemon Unix socket，但只能在**与 daemon 同机**的 shell（SSH 进 GCP VM）跑。这是 D-10 / §11.U 已锁的 raw-mode TTY 入口，跟 desktop 是不同 surface。

**含义**：

| 客户端 | transport | 必经 server |
|---|---|---|
| `tether attach <sid>` (在 daemon 同机) | UDS | ❌ 直连 daemon |
| Desktop (笔记本) | WT-over-HTTP/3 | ✅ |
| Mobile (手机) | WT-over-HTTP/3 | ✅ |

**为什么不留 UDS 直连分支**：
1. **现实部署**：daemon 跑用户 GCP VM 是实际场景；笔记本本地跑 daemon 在 v0.1 不是目标用户路径
2. **代码简洁**：单 transport 路径减 ~80 LOC + 整套 probe / fallback / 错误处理
3. **架构对称**：desktop 跟 mobile 行为完全一致，调试 / 文档 / 测试矩阵都减半
4. **未来扩展**：将来真有"daemon + desktop 同机"用例（dev mode 本地试用），可以加 UDS 路径——但 v0.1 不预先做

#### 11.Y.6 与 §11.X / D-18 的关系

D-18 9 项 forward-compat 预留中，#5（App parseContent → renderBlock pipeline）从原"v0.1 ~50 LOC stub + v0.2 完整实现"**升级为 v0.1 1st-class 完整实现**。其他 8 项不变。

**为什么升级**：D-19 锁定后，App UI 三个 layout 容器都依赖 fenced block 渲染——v0.1 不实现就退化成裸 markdown chat（A 选项），整个「skill 是灵魂」的 vision 落不下来。

**v0.2 还要做什么**：streaming FSM（部分发送的 block 增量 patch）、blob proxy 完整实现（v0.1 仅 path 占位）、状态重放协议、按钮 → slash command 安全 audit chain——这些 v0.1 不做，但渲染器 / shell 层 v0.1 已 ship。

### 11.Z Skill 模型 ✅ **已锁定（D-20，cc plugin + `tether.toml` overlay）**

**核心决定**：tether skill 不发明新格式，直接 = **cc plugin 标准结构 + 一个补充 `tether.toml`**。daemon 不管 skill 进程生命周期——全部 delegate 给 cc 的 plugin loader + hook 系统。

#### 11.Z.1 Skill 目录结构（与 cc plugin 一致）

```
<global-pool>/<skill-name>/
├── .claude/                    # cc plugin 标准
│   └── plugin.json             # cc plugin manifest（cc 强制）
├── tether.toml                 # tether-specific 元信息（可选）
├── skills/
│   └── <name>/
│       ├── SKILL.md            # markdown 上下文，cc 加载
│       └── references/         # 长文档，按需 read
├── hooks/
│   └── hooks.json              # cc hook event → 脚本映射
├── commands/                   # slash command markdown
├── scripts/                    # hook / command 调用的 shell/Python/Node 脚本
├── agents/                     # sub-agent markdown definitions
└── README.md
```

**没有 `tether.toml` 的 skill** = 普通 cc plugin，仍可装可用（可移植性最大化），但拿不到 tether-specific 功能（fenced block pre-register / mobile hint / state 协议）。

#### 11.Z.2 `tether.toml` schema（v0.1 草案）

```toml
# 必填 - skill 身份
[skill]
name = "dag"
version = "0.1.0"
description = "DAG-driven multi-step planning + execution skill"

# 必填 - agent 兼容声明（v0.1 仅 cc）
[skill.agents]
primary = "cc"                          # 这个 skill 默认走哪个 agent
tested = ["cc"]                         # 作者实测过的 agent target
                                        # v0.2 可加 ["cc", "codex"]——表示作者验过 codex adapter

# 可选 - 该 skill 会 emit 什么类型的 fenced block（让 App 端 pre-register renderer）
[tether.fenced_blocks]
emits = ["dag", "media"]                # 引用 §11.AA / 2026-04-27-phase1-dag-protocol.md §3 四类 block
preferred_layout = "full"               # 默认偏好桌面中栏全屏（mobile_default 由 App 端按 viewport 推导）
mobile_default = "compact"

# 可选 - 移动端 UI hints
[tether.mobile]
default_card_layout = "compact"
quick_actions = ["rollback", "approve"]  # mobile 上把这些 verb 抬到 card 主操作位

# 可选 - 状态文件协议（D-18 #1 workspace_dir）
[tether.state]
workspace_dir = ".tether/dag/"          # 写到 <workspace>/.tether/dag/
file_lock = true                        # 多 cc session 同时跑时是否需要 file lock（v0.1 用 flock）
gitignore_default = true                # tether spawn 自动加 .tether/dag/ 到 workspace .gitignore

# Hook event 命名（v0.1 = cc 原生，不做 namespace 化；详见 §11.Z.5）
# hooks/hooks.json 直接用 cc 标准 event 名（PreToolUse / PostToolUse / SessionStart 等）
# v0.2 加 opencode 时再决定是否引入 namespace；当前 polyforge 16 个 skill 不改一行能用
```

#### 11.Z.3 执行模型 — daemon 不感知 skill

```
┌────────────────────────────────────────────────────────────┐
│ tether daemon                                              │
│                                                            │
│  1. install: symlink/copy <global-pool>/<skill> →          │
│     <workspace>/.claude/plugins/<skill>                    │
│  2. spawn cc with plugin enabled                           │
│  3. jsonl_watcher 看 cc 输出                               │
│     ├─ fenced block (dag/form/candidates/media) → wire    │
│     │  envelope output.agent-event → App 渲染              │
│     └─ 普通 markdown → 同样转发，App 当 chat 文本           │
│                                                            │
│  daemon 完全不知道 "dag skill" 是什么——只看 cc 输出 JSONL │
└────────────────────────────────────────────────────────────┘
                             │
                             ▼
┌────────────────────────────────────────────────────────────┐
│ cc 进程（cc 自己 spawn skill hook 脚本）                    │
│                                                            │
│  • cc 读 SKILL.md → 上下文                                 │
│  • cc 触发 hook event（PreToolUse 等 cc 原生 event 名）→  │
│    cc 自己 spawn                                            │
│    hooks/hooks.json 里指定的 shell/Python script           │
│  • script 写 stdout fenced block → cc 把 block 写进 JSONL  │
│  • script 调用 `tether send-input "<text>"` 给 cc 发指令   │
│    → 走 daemon CLI → input gate (D-09) → cc PTY            │
└────────────────────────────────────────────────────────────┘
```

**关键性质**：
1. daemon 跟 skill **没有直接 RPC 接口**；耦合面只有「fenced block schema」（§11.AA / `2026-04-27-phase1-dag-protocol.md §3`）
2. skill 进程归属 cc，不是 daemon——cc 的 plugin sandboxing / 资源限制全部继承
3. 换 skill 对 daemon 是 **0 改动**；daemon 只看 cc 输出
4. skill → agent 写：通过 `tether send-input` CLI，复用 input gate (D-09)
5. skill ↔ skill v0.1 不直接通信（各自独立运行；通过 workspace 共享文件状态间接交互）
6. **daemon 缓存 fenced block 渲染快照**（详见 §11.Z.9）：每 (sessionId, skillName, blockId) 缓存最近 envelope，App attach 走 catch-up channel 拉取——**daemon 不读 skill state file**

#### 11.Z.4 Adapter 模块（v0.2 多 agent 准备）

```
internal/skill/adapter/
├── cc.go              ← v0.1 唯一真实现（~70 LOC）
│                        pure symlink: <global-pool>/<skill> → <workspace>/.claude/plugins/<skill>
│                        解析 tether.toml metadata 写到 daemon 内部 registry（fenced block emits 等）
│                        cc plugin manifest 由 skill 仓库自带，adapter 不生成
├── codex.go           ← v0.1 stub（panic("not impl in v0.1")），v0.2 ~200 LOC
│                        textually rewrite skills/ verbatim, generate .codex-plugin/plugin.json
├── opencode.go        ← v0.1 stub，v0.2 部分实现（commands/agents 互通；hooks 需作者另写 .ts）
├── aider.go           ← v0.1 stub，v0.2 lossy adapter（concat SKILL.md 进 CONVENTIONS.md，hooks 丢弃 + warn）
└── cursor.go          ← v0.1 stub，v0.2 lossy adapter（concat 进 .cursor/rules/）
```

**为什么 day 1 就把 stub 命名清楚**：跟 D-17 AgentProvider seam 同纪律——把"未来要做什么"显式写成空文件，避免设计阶段重复扯"v0.2 怎么办"；同时阻止有人把 cc-specific 逻辑塞进 `internal/skill/install.go` 通用路径。

#### 11.Z.5 Hook event 命名（v0.1 = cc 原生，不做命名空间化）

**v0.1 决议**：skill 的 `hooks/hooks.json` 直接使用 cc 标准 event 名（`PreToolUse`、`PostToolUse`、`SessionStart`、`UserPromptSubmit`、`Stop`），**不引入** `cc.` 前缀或命名空间机制。

**为什么不做（YAGNI）**：
1. v0.1 是 cc-only 的（D-17）；命名空间化解决的是 v0.2 多 agent 的问题，现在不存在
2. polyforge 现有 16 个 skill 的 `hooks.json` **不改一行能装到 tether**——D-20 主卖点；引入 `cc.` 前缀会破坏这个承诺
3. v0.2 真要支持 opencode 时一定会重新评估 hook 模型——opencode 的 hook 是 **TS 代码 + 自定义 event 词汇**（review 2026-04-28 调研结论），不是 event-name 重命名能解决；届时已有 skill 要写新的 `opencode-hooks.{ts,json}` 配套，namespace 化提前抽象大概率错
4. 简化 cc.go adapter：从复制 + rewrite 退化为**纯 symlink + 解析 `tether.toml`**，~70 LOC 而非 ~150 LOC

**v0.2 加 opencode 时怎么办**（不在 v0.1 范围，仅记录意图）：
- skill 仓库可同时存在 `hooks/hooks.json`（cc 用）+ `hooks/opencode.ts`（opencode 用，TS 代码 per opencode 模型）
- 或者那时引入新的 namespace 机制（届时按需设计）
- 当前不约束 v0.2 选择

#### 11.Z.6 Skill 安装 / discovery 流（v0.1）

```bash
# 用户安装
tether skill install dag                       # 走 §11 Q8 blessed list（main repo skills.toml）
tether skill install https://github.com/x/y    # 任意 git URL 直装

# 落点
~/.tether/skills/dag/                          # 全局池（§11 Q7）

# workspace 启用（写 workspace tether.toml，v0.1 单文件 manifest）
<workspace>/tether.toml:
  [skills]
  enabled = ["dag", "writing-plans"]

# tether spawn <workspace> 时自动：
#   1. 读 workspace tether.toml [skills].enabled
#   2. 对每个 enabled skill 调 cc.go adapter:
#      symlink ~/.tether/skills/dag → <workspace>/.claude/plugins/dag
#   3. 启 cc，cc plugin loader 看到 .claude/plugins/dag 自动加载
```

#### 11.Z.7 LOC 估算

| 模块 | LOC |
|---|---|
| `internal/skill/manifest.go`（tether.toml 解析）| ~80 |
| `internal/skill/install.go`（global pool + git clone + blessed list resolve）| ~150 |
| `internal/skill/adapter/cc.go`（v0.1 唯一真实现，pure symlink）| ~70 |
| `internal/skill/adapter/{codex,opencode,aider,cursor}.go`（stub）| ~50 合计 |
| CLI `tether skill {install/list/remove/info}` | ~120 |
| **合计 v0.1 skill 系统** | **~470 LOC** |

跟 daemon goroutine（~1330-1430 LOC）+ App UI（~3000 LOC，§11.Y.3 重估后含 a11y / i18n / pair / settings / catch-up / error 全套）相比是小头，但「skill 是灵魂」承诺的兑现就在这 470 行。

#### 11.Z.8 v0.1 不做什么（明确不在范围）

1. **skill 沙箱**——继承 cc 自己的（cc plugin 跑用户 shell 脚本天然不沙箱；用户责任）
2. **skill 间通信 RPC**——多 skill 通过 workspace 文件状态间接共享
3. **skill 版本依赖求解器**——v0.1 不支持 skill 依赖其他 skill
4. **skill marketplace UI**——`tether skill list` CLI + 主 repo blessed list 即可，无 web marketplace
5. **skill hot-reload**——改 skill 代码后要 `tether respawn <session>` 才生效（cc plugin loader 限制）
6. **opencode/codex/aider/cursor adapter 真实现**——v0.2 / v0.3 排期

#### 11.Z.9 Skill 状态文件归属 ✅ **已锁定（B-4=A，skill 自管 + daemon 仅缓存渲染快照）**

**核心原则**：daemon **不读** skill state file；daemon 只缓存最近 emit 的 fenced block（"渲染快照"），跟 skill 内部状态完全解耦。

**两层状态分离**：

| 层 | 谁拥有 | 内容 | 大小级 | 谁能看 |
|---|---|---|---|---|
| **Skill 内部 state**<br/>`<workspace>/.tether/<skill>/state.json` | skill 脚本 | 完整逻辑状态（log / 路径列表 / rollback history / checkpoint）| 1KB-50MB | 只 skill 脚本 |
| **daemon 渲染快照**<br/>daemon RAM + SQLite | daemon | 最近 N 个 fenced block（"给用户看的精简可视化"）| 总 < 10MB | App（catch-up 时拿）|

**Skill 自管的具体规则**：
1. skill 脚本读写 `<workspace>/.tether/<skill-name>/`（路径在 `tether.toml` `[tether.state] workspace_dir` 声明）
2. skill 自己负责原子性（temp file + rename pattern）
3. `[tether.state] file_lock = true` 时 skill 写前 `flock` workspace_dir/lock
4. `[tether.state] gitignore_default = true` 时 `tether spawn <workspace>` 自动 ensure `.tether/` 在 workspace `.gitignore` 里
5. workspace 删除时 `.tether/` 一起删（无独立 cleanup）

**daemon catch-up 缓存机制**（属 §3.3 wire 层）：
- daemon 内部 LRU：`map[(sessionId, skillName, fenceSeq)] -> latest_envelope`，其中：
  - `sessionId`：envelope plaintext 已有
  - `skillName`：jsonl_watcher 从 fence tag 后缀提取（`^```\w+:[\w-]+$` 正则匹配 §11.AA.1 语法）写入 envelope plaintext metadata
  - `fenceSeq`：daemon 内部 monotonic 计数器，per (sessionId, skillName)，每个新 fence block 自增 1
- **关键**：daemon **不**用 block 内部 `instance_id` 字段做 key——那个字段在 ciphertext payload 内，daemon 不解密读不到。`fenceSeq` 是 daemon-side 替代，按 fence 出现顺序编号，对 catch-up replay 顺序还原已经够用
- App 解密后用真正的 `instance_id` 做组件去重 / 跨 skill 状态合并；daemon 的 `fenceSeq` 仅是 LRU 槽位标识，对 App 不可见
- 上限：每 (session, skill) 最多 20 个 block snapshot（fenceSeq 模 20 wrap-around，旧 snapshot 被新的同 seq 覆盖）；全局上限 1000 blocks 或 10MB（先到先 evict）
- App attach 进来时 catch-up channel 按 fenceSeq 升序 replay 这些 snapshot envelope（直接重发，**不解密**——envelope 是 ciphertext）
- D-18 #9 reserve 的 `catchup.state-snapshot` envelope kind 即此机制承载
- LOC ~80（LRU + emit/replay 钩子 + fence tag 正则解析 + per (session,skill) 计数器 ~15 LOC）

**为什么不用 `instance_id`** （v0.2 phase 1 spec §3 block schema 字段）：
- 加密约束（D-12）：daemon 看不到 plaintext payload，所以 `instance_id` 这个 schema 内字段对 daemon 不可见
- App 端有完整 plaintext 可正常用 `instance_id` 做组件 reconcile（不冲突，daemon 和 App 用各自的 key 互不干扰）

**review 2026-04-28 P2 #16** 提到的 `blockId` vs `instance_id` 命名漂移已统一：daemon LRU = `fenceSeq`（完全 daemon-side 概念）；block schema = `instance_id`（plaintext payload 字段）。两个名字两个意义，不再混用。

**为什么 daemon 不读 state file**：
1. 兑现 D-20「daemon 不感知 skill」承诺；skill state schema 怎么演化都不影响 daemon
2. State file 可能 5-50MB，走 wire 是带宽灾难；可能含 absolute path / 内部 ID 不该暴露给 App
3. App 永远跟 fenced block 这个稳定 schema 耦合，不跟 skill 内部 schema 耦合

#### 11.Z.10 Active skill 切换语义 ✅ **已锁定（B-5=β，多 skill 同时 active + 时间轴混排）**

**核心决定**：workspace 里所有 enabled skill **同时 active**，没有"模式切换"概念；fenced block 按时间序混排进 chat。

**实际行为**：
1. workspace `tether.toml` 里 `enabled = ["dag", "writing-plans", "media-review"]` → 全部 symlink 到 `<workspace>/.claude/plugins/` → cc 全部加载进上下文
2. 任何 skill emit 的 fenced block 都直接入 chat 流
3. 每个 fenced block 通过 **fence tag 后缀语法**带 skill 身份：` ```<block-type>:<skill-name> `（详见 §11.AA.1）。daemon 在 fence tag 行 grep 出 `<skill>`，写入 envelope plaintext metadata；block payload 内**不再**重复携带 skill 字段。App 收 envelope 即知 block 归属哪个 skill，无需解密 body
4. App 默认按时间序混排所有 block；可选 filter chip 只看某个 skill
5. **没有"切换 active skill"的全局状态机**；不存在"当前激活的 skill"概念

**桌面三栏 + 手机 layout 行为**：

| 容器 | 默认 |
|---|---|
| 桌面中栏 | 时间轴混排所有 block；顶 filter chip 可只看某 skill；点单 block → 全屏详情 |
| 桌面右栏 chat | inline card 时间序，每张 card 左上角小 icon 标 skill |
| 手机 chat 流 | 同桌面右栏；filter chip 在顶栏抽屉 |

**为什么 β 而不是 α（单 active）**：
1. cc plugin 多载是天然行为；强行单 active 跟 cc 行为打架
2. 真实场景多 skill 并行：写代码 + 跑 dag + 看 form 询问，全部混在 chat 流是用户期待
3. polyforge 现有 16 个 skill 同时 enable 已经在用，模型一致

**Skill 作者承诺**：每次 emit fenced block 必须用 **fence tag 后缀语法**写 ` ```<block-type>:<skill-name> `（详见 §11.AA.1；SKILL.md 模板会教这个规范）。daemon 通过正则 `^```\w+:[\w-]+$` 提取 fence 行的 `<skill-name>` 写入 envelope plaintext metadata，**不进 block body**——保留 D-20 「daemon 不感知 skill」承诺。author 漏写 `:skill` 后缀时 → 降级为 "unknown skill"（§11.AA.1 已规范），仍渲染但 filter / icon 退化。

#### 11.Z.11 Skill 缺失 fallback ✅ **已锁定（B-6=C default + opt-in flags）**

**场景**：用户 clone 一个 workspace，里面 `tether.toml` `[skills].enabled = ["dag", "media-review"]`，但本地全局池 `~/.tether/skills/` 没装 `media-review`。`tether spawn <workspace>` 怎么办？

**默认行为 = C（fail with helpful error）**：

```
$ tether spawn ~/work/xiyou
error: workspace requires skills not in global pool:
  - media-review (enabled in tether.toml)

install with:
  tether skill install media-review

or to spawn with available skills only:
  tether spawn --allow-missing ~/work/xiyou
```

daemon 不 spawn cc，exit 非零。**不 implicit 网络调用**——supply chain 攻击面闭合在用户主动决定。

**Opt-in flags**（用户显式承担风险）：

| flag | 行为 |
|---|---|
| `tether spawn --auto-install` | daemon 自动走 §11 Q8 blessed list + git URL clone 缺失 skill 到全局池，**不**交互 prompt；clone 失败回到 C 错误信息 |
| `tether spawn --allow-missing` | 跳过缺失 skill，仅用已装的 skill spawn cc；stderr 打 warning + missing 列表；workspace 行为可能跟 manifest 不符 |

**解析顺序**（默认 C）：
```
1. 读 workspace tether.toml [skills].enabled
2. for each skill name:
   - 查 ~/.tether/skills/<name>/ 是否存在
3. 缺失任意一个 → 打印列表 + 修复命令 → exit 1
4. 全在 → 进 cc.go adapter symlink + spawn cc
```

**LOC**：~30（resolveSkills + 错误格式化）

#### 11.Z.12 边界 case 处理（v0.1 明确）

| # | 场景 | 处理 |
|---|---|---|
| 1 | 多 cc session 共享 workspace | state file per-skill **不**per-session；同 workspace 两 cc session 改同一 dag → `[tether.state] file_lock = true` 强制 flock；daemon 不参与 |
| 2 | 跨 skill 状态读 | v0.1 允许（`.tether/` 下无 namespace 隔离）；skill 作者自治；v0.2 真有问题再加权限 |
| 3 | State file mobile 同步 | **不同步**。mobile 永远拿 daemon 缓存的 fenced block 快照，不拿 raw state file |
| 4 | Block snapshot LRU 容量 | 每 (session, skill) 20 个；全局 1000 blocks 或 10MB；LRU evict |
| 5 | Block snapshot 加密 | daemon 缓存的是 envelope 整体（含 ciphertext），不解密；E2E 性质保留——server-compromise confidentiality 同 D-12 |

#### 11.Z.13 Runtime ownership model ✅ **已锁定（review 2026-04-28 P1 #12 = D，tether 拥有 cc runtime）**

**核心 mental model**：tether 不是"在用户已有 cc 环境上加 skill 的工具"——tether **拥有** cc runtime，runtime 启动时零 skill，所有 skill 经 `tether skill install` 进入。

**Production 假设**（未来商业化每用户独立容器时）：
- cc 跑在 tether 提供的容器（docker image / k8s pod）里
- 容器初始**无 `~/.claude/plugins/`**——零 user-level skill
- 唯一入口：tether daemon spawn cc
- 无"裸 cc"调用路径
- 用户操作 100% 经 `tether skill install` / `tether spawn` 等 CLI

**v0.1 dev 模式假设**（当前阶段，用户在自己机器跑 tether）：
- 用户机器上可能已有 `~/.claude/plugins/`（如 polyforge）——这是**用户自己的事**，tether 不管
- cc 默认行为是 merge user-level + workspace-level plugins → tether 启用的 skill 跟用户 user-level 已有 plugin **并存加载**
- 视为 **过渡期限制**，不是 tether 设计问题
- v0.1.x / 商业化引入容器化运行环境时，user-level concern 自然消失

**4 个边界 case 的答案**（来自 review P1 #12）：

| 问题 | Production（容器）| v0.1 Dev（自己机器）|
|---|---|---|
| Skill 重名 | 不可能（user-level 不存在）| cc 默认 merge 行为，用户接受 |
| `tether kill` symlink 清不清 | 容器 ephemeral，无所谓 | 不清（避免误删用户手动操作）；workspace 删除自然清 |
| 裸 cc 调用 workspace | 不可能（无裸 cc 入口）| 用户自找麻烦，hook 失败 graceful degrade + warning |
| enabled vs user-level skill | 仅 enabled 生效 | dev 默认 cc merge，用户接受 |

**实现影响**（vs 我之前的 hands-off A 选项）：
- **代码相同**：cc.go adapter pure symlink global pool → workspace `.claude/plugins/<skill>`，~70 LOC
- **不引入** `--no-user-plugins` flag（会破坏 v0.1 dev 工作流）
- **不引入** `tether spawn --clean`（v0.1 不需要；workspace 删除已经够）
- 文档明确"production 假设独立 runtime；dev 假设用户自管 user-level"

**未来 polyforge 怎么进 tether 生态**：v0.2+ 把 polyforge 包装成 tether skill（`tether skill install polyforge` 装一个 meta-skill，里面 16 个 sub-skill）。这是 polyforge 维护者后续的事，不在 v0.1 范围。

**v0.1.x 容器化路径**（独立 milestone）：
1. 提供 `gmicloud/tether-runtime` Docker image（cc + tether-cli + 空 user-level）
2. 用户 `tether server` 时启动容器跑 daemon
3. user-level 物理隔离 → production 模型成立

### 11.AA Fenced block 协议引用 ✅ **锚定（v0.1 1st-class，规范文档 = 2026-04-27-phase1-dag-protocol.md §3）**

**目的**：v0.1 D-19 把 fenced block 协议（dag / form / candidates / media 四类 block）从 v0.2 提前到 1st-class，但完整 schema 规范不在本文档内——位于 companion spec `2026-04-27-phase1-dag-protocol.md` §3。本节是 **canonical reference anchor**，给本文档其他章节统一引用。

**v0.1 实现 vs v0.2 实现**：

| 部分 | v0.1 | v0.2 |
|---|---|---|
| 4 类 block 基础 schema（`v` / `instance_id` / kind 字段）| ✅ 完整实现 | 同 v0.1 |
| App 端 4 类 renderer（DagBlock / FormBlock / CandidatesBlock / MediaBlock）| ✅ 完整实现，桌面 + 手机共用 | 同 v0.1 + 增量优化 |
| Streaming FSM（部分 block 增量 patch）| ❌ v0.1 不做（block 整段 emit + 整段渲染）| ✅ 完整实现 |
| Blob proxy（`/blob/<sha256>` server 路由）| ❌ 仅 path 占位 + stub 501（D-18 #3）| ✅ 完整实现 |
| 状态重放（block 增量补丁）| ❌ daemon 缓存最近完整 envelope LRU（§11.Z.9）| ✅ 完整 patch chain replay |
| Button → slash command 安全 audit chain | ❌ v0.1 简化版（同手打记录）| ✅ envelope ID 链全审计 |

**v0.1 1st-class 的"完整实现"含义**：能正确**整段** emit + 渲染 4 类 block；不需要 streaming patch；不需要 blob proxy；不需要 patch-replay。已经够 D-19 三栏 / chat-first 体验。

#### 11.AA.1 Fence tag 语法扩展 ✅ **已锁定（review 2026-04-28 P0 #2 决议 = β'，fence tag 后缀 `<type>:<skill>`）**

**核心扩展**：fenced block 的 language tag 允许 `<block-type>:<skill-name>` 后缀，把 skill 身份信息**结构上 co-locate** 在 fence 行，**block payload 内不再放 skill 字段**。

```markdown
当 dag skill emit dag block：
```dag:dag
v: 1
instance_id: n3
nodes: [...]
```

当 xiyou-animation skill emit media block：
```media:xiyou-animation
v: 1
url: tether://workspace/xiyou/ep03/f001.png
```

当 writing-plans skill emit form block：
```form:writing-plans
v: 1
fields: [...]
```
```

**语法定义**：
```
fence-tag := <block-type> [ ":" <skill-name> ]
block-type := "dag" | "form" | "candidates" | "media"
skill-name := [a-zA-Z0-9_-]+      // 与 §11.Z.6 skill 安装命名约束一致
```

- **必带 `:skill` 部分**：v0.1 标准。但如果 skill author 漏写或第三方 skill 没接 tether overlay → degrade 成 `unknown` skill，仍然渲染（§11.Z.10 已声明"降级"行为，对齐）
- **不影响 markdown 标准**：fence tag 在 markdown spec 里就是 free-form info string，`dag:dag` 跟 `python` 同性质合法

**为什么选 β' 而非 β（block payload 内放 `skill` 字段）**：

| 维度 | β (payload 内字段) | β' (fence tag 后缀) ⭐ |
|---|---|---|
| daemon 是否 parse block 内容 | 是（要抽 `skill` 字段）| **否**（只 grep fence 行）|
| D-20「daemon 不感知 skill」承诺 | 微违反 | 完整保留 |
| Encryption 兼容 | 必须把 skill 字段从密文 payload **复制**到 envelope plaintext，redundant | fence tag 本来在 envelope plaintext metadata，无 redundancy |
| Skill author DX | 每个 block 都要在 yaml 里写 `skill: <name>` 一行 | fence tag 一处写完 |
| App renderer dispatch | 解密 block 后读 yaml | 直接看 envelope metadata（已有 fence tag 解析）|
| 向后兼容（`\`\`\`dag` 无 skill）| 缺字段 → degrade unknown | 缺 `:skill` → degrade unknown（同语义）|

**daemon 处理逻辑**（§11.Z.9 daemon catch-up LRU 的实际实现）：

```go
// jsonl_watcher 解析 cc 输出 markdown
func parseFenceTag(line string) (blockType, skillName string, ok bool) {
    // 匹配 ^```<type>(:<skill>)?$
    matches := fenceTagRegex.FindStringSubmatch(line)
    if matches == nil { return "", "", false }
    blockType = matches[1]
    if len(matches) > 2 && matches[2] != "" {
        skillName = matches[2]
    } else {
        skillName = "unknown"  // §11.Z.10 degrade 行为
    }
    return blockType, skillName, true
}

// daemon 不进 block body 一行
func handleBlock(env *Envelope, blockType, skillName string) {
    env.Plaintext.BlockType = blockType
    env.Plaintext.SkillName = skillName
    // body 直接进 ciphertext，daemon 不读
    routeAndCache(env)
}
```

**Privacy caveat**：skill name 走 envelope plaintext metadata，server 可见。按 D-12 现有 threat model（"server-compromise confidentiality"——不是完整 forward secrecy；providerType / sessionId / kind 都已经明文），skill name 是同级别 routing metadata，可接受。**v0.1 不做**额外的 metadata 隐藏（onion routing 之类 v0.1.x+ 再说）。

**Companion spec 待补**：`2026-04-27-phase1-dag-protocol.md` §3 当前只定义 block payload schema，**未规范 fence tag**。本决议锁定后需在 companion spec §3 头部加一节 "Fence tag 语法扩展"，引用本节 §11.AA.1 作为权威定义；4 类 block 各自的示例（§3.1 / §3.2 / §3.3 / §3.4）的 fence tag 全部更新为 `<type>:<skill>` 形式。**这个 patch 在锁定后立即做（一次性 ~10 行 markdown 改动）。**

**所有引用本协议的章节统一指向**：本节 §11.AA + `2026-04-27-phase1-dag-protocol.md §3`（block schema）+ companion spec `§3.5 / §4`（streaming + parser，v0.2 才完整，v0.1 整段路径足够）。

---

## 11.Δ Review 延后清单（2026-04-28 spec review 后未阻塞实施的项）

下面这些是 2026-04-28 review 提出但**不阻塞 v0.1 实施开始**的项。按"何时必须解决"排期：

### 实施期内自然解决（不阻塞开搞）

| # | 项 | 何时 |
|---|---|---|
| P1 #6 | PoC-2.6 Android smoke test：`web-transport-quinn` × `aarch64-linux-android` 编译 + 真机 / emulator WT dial（桌面常规编译不算 spike）| Tauri build matrix 那 2.5 周开始前（实施第 ~9 周）|
| P1 #11 | Pair flow UX wireframes | App UI 5 周内（已含在 ~350 LOC 估算）|
| P2 #14 | v0.2 phase 1 spec §5.4 用 `\n` 跟 F-01 `\r` 矛盾 | App UI 实施期改 companion spec |
| P2 #16 | `blockId` vs `instance_id` 命名漂移 | App UI 实施期统一为 `instance_id` |
| P2 #17 | Tauri 4-target 1.5 周偏紧 | 真做时如果 hit 难点，buffer 1-2 周吸收 |
| P2 #18 | Workspace tree 50 ws / 1000 文件 virtualization | 已含在 `<WorkspaceTree />` 180 LOC 估算 |

### 公开发布前 sprint 解决

| # | 项 | 何时 |
|---|---|---|
| P1 #8 | 本地 dev loop（loopback HTTPS）让 contributor 不必 GCP VM 也能跑全栈 | Repo 公开前 1 周 |
| P1 #9 | `--auto-install` URL 源（blessed list manifest schema）| Repo 公开前 1 周；或砍 `--auto-install` flag（B-6 默认 fail-closed 已够）|
| P1 #10 | License (Apache 2.0 推荐) + 品牌名查 npm/github + repo public 时机（推 γ：dogfood private + ship 前 1-2 周转 public）| First public commit 前 |
| P2 #19 | 数据备份 / GCP VM 丢失恢复策略 | 公开前文档化（v0.1 用户手册一部分）|

### v0.1.x 或后续 milestone

| # | 项 | 何时 |
|---|---|---|
| P2 #15 | session_key idle rotation（防长期不关 cc session 持续累积明文风险）| v0.1.x 加默认 24h idle rotation |
| P2 #13 | D-8（B-8 brainstorm letter）vs D-21 顶层 ID 含义 collision | 文档美容；v0.1.1 整理时改 |

### 已锁定不再讨论

| # | 项 | 决议 |
|---|---|---|
| P0 #1 | §11.U 错引 fenced block 协议 | ✅ 加 §11.AA 锚点 + 错引修正 |
| P0 #2 | `skill: <name>` 字段位置 | ✅ β' fence tag 后缀 ` ```<type>:<skill> ` |
| P0 #3 | §5.0 残留 `tether spawn --agent codex` / `tether agents` | ✅ 删掉，与 D-17 对齐 |
| P0 #4 | Hook namespace 重写位置 | ✅ c YAGNI，v0.1 完全不做命名空间化 |
| P1 #5 | Android transport native vs polyfill | ✅ **ii-rebuilt**（review re-decision 2026-04-28）：tether 自写 Tauri command 包 `web-transport-quinn`，跑遍 Android + macOS + Linux + Windows 4 个 target；放弃 abandoned `tauri-plugin-web-transport`；timeline +1 周（14-16 周）|
| P1 #7 | App UI 780 LOC 不切实际 | ✅ X 全 surface 重估 ~3000 LOC，timeline 13-15 周 |
| P1 #12 | Polyforge 共存 | ✅ D tether 拥有 cc runtime（production 容器隔离 / dev 容忍 user-level）|

---

## 12. 已经讨论过的备选方案及驳回理由（不再开放）

记录在册，避免日后重开旧账。

### 12.A 用 SDK 调 cc
- **场景**：happy 在 remote 模式下用 `@anthropic-ai/claude-agent-sdk`
- **驳回理由**：
  - 设计原则 #3 明确 "spawn 命令而非 SDK"
  - 用 SDK 就回到 happy 的"切换 = 杀进程重启"模型
  - 失去切换瞬间无感的核心体验
- **影响**：CLI 永远不通过 SDK 跟 cc 通信；mode ① + ③ + JSONL 唯一通路

### 12.B 用 Rust 替代 Go
- **驳回理由**：
  - 技术上等价（quinn ≈ quic-go，portable-pty ≈ creack/pty）
  - 无 server-CLI 同语言共享 wire schema 的便利
  - 编译速度 / 学习曲线 / 生态友好度 Go 更适合单人节奏
- **如果未来反悔**：Rust 端口可行，但成本是重写一次

### 12.C 用 Node.js 实现
- **驳回理由**：
  - 不用 SDK 后 Node 没有结构性优势
  - node-pty native 模块编译有平台相关风险
  - 单二进制分发对自托管用户体验差距巨大

### 12.D tmux 当 cc 的容器
- **驳回理由**：
  - 用户实测 tmux 跑 cc 启动失败（具体症状暂不可考，但事实上不工作）
  - tmux 的"智能"终端模拟会篡改 cc 的现代 escape（OSC 8、kitty graphics、true color）
  - 即使能跑，依赖 tmux 安装（容器镜像里要 +20MB）

### 12.E abduco / dtach 当 cc 的容器
- **场景**：daemon 死 cc 不死的"哑容器"方案
- **驳回理由**：
  - 设计 D-07 已接受 "daemon 死 = cc 死，靠 resume 兜底"
  - 加 abduco 是为了避免这个问题，但用户接受了这个 tradeoff
  - 减少一个外部依赖
- **如果未来反悔**：可以以"加固"姿态在 v0.1.1 引入，daemon 优先连 abduco socket，没有就 fallback 到自持 PTY

### 12.F Mobile App 直连 daemon（两层拓扑）
- **驳回理由**：
  - 用户原始设想里就有 server 这一层
  - 三层解决 NAT、离线 catch-up、Push 通知规范化
  - 给 v0.2 多机器/多用户预留路径

### 12.G "终端入口必须能用 native cc"
- **驳回理由**：
  - cc 自己的 session 模型是 1 进程 1 文件，不支持多端
  - 真用 native `cc` 等于绕过 daemon，lock 机制全废
  - `tether attach` 等价用户体验，工程量可控（~500-700 LOC Go）

---

## Appendix A：2026-04-26 讨论过程（决策日志）

按时间顺序记录关键决策点，便于后续接续：

1. 整体目标：基于 happy 的心智模型评估，确定三个核心架构选择（QUIC + Go + 不用 SDK）
2. 反复阅读 happy 的 6 个 packages 源码作为 prior art 参考
3. 讨论 daemon-cc ownership 关系：明确 attach 模型 vs native cc 不可行
4. 解释"为什么不能直接 cc"：cc 自己不支持 multi-attach，session 文件 1 进程 1 用
5. 讨论用 tmux 当 cc 容器：用户实测 tmux + cc 启动失败 → 排除（12.D）
6. 讨论用 abduco/dtach 当容器：分析后用户接受 daemon-死=cc-死 → 不引入容器（12.E）
7. 决定整体语言：Go + 不用 Rust + 不用 Node（12.B/C）
8. 解释 Go 怎么跟 cc 交互（PTY ① + Hooks ③ + JSONL）
9. 用户提出 watchdog 模式（main + daemon goroutine + client goroutine）：架构清晰起来
10. 用户澄清 client goroutine 是连接 server 的 → 拓扑变 3 层
11. 写下本文档

后续的讨论开始顺序建议：
- ~~§11.C E2E 加密层级~~ ✅ 已锁定 2026-04-27：C1 精简版（X25519 + XChaCha20-Poly1305 + HKDF；device-pair → ephemeral session_key 两层；~390 LOC Go），D-12
- ~~§11.A WT vs WT+WS、§11.T raw QUIC vs HTTP/3~~ ✅ 已锁定 2026-04-27：纯 WT、CLI 同走 WebTransport-over-HTTP/3
- ~~§11.G + §11.V Mobile 壳子~~ ✅ 已锁定 2026-04-27：**Tauri Mobile from v0.1**（不出 PWA），D-13。**注**：原始决议提到 iOS WT 走 `tauri-plugin-web-transport`；2026-04-28 review P1 #5 重新决议为 **ii-rebuilt = tether 自写 Tauri command 包 `web-transport-quinn`**，跨四平台统一，废弃 `tauri-plugin-web-transport`
- ~~§11.M 加密原语~~ ✅ 已锁定 2026-04-27：随 §11.C
- ~~§11.B 离线消息暂存~~ ✅ 已锁定 2026-04-27：B4 + session_key 跟 cc session 等长（修 §6.1）+ 暂存满 trim agent-bytes
- ~~§11.E Push 通知~~ ✅ 已锁定 2026-04-27：E2（CLI 自推 APNs/FCM）+ Pusher 多态抽象（**翻文档原 E1**）
- ~~通信模型 / wire 协议~~ ✅ 已锁定 2026-04-27（D-14）：完整 envelope schema + 6 类 operation taxonomy + 5 通道 stream 分配（control / events / agent-bytes / catch-up / datagram）+ chunk 16KB/100ms + trim 保 events 丢 agent-bytes + 三个核心 flow walkthrough。详见 §3.3
- ~~§11.D Lock 状态机细节~~ ✅ 已锁定 2026-04-27（D-15）：单层 lock + 60s auto release + force takeover；view-only v0.1 不做
- ~~§11.U 远程 attach 入口~~ ✅ 已锁定 2026-04-27：U1（仅 Unix socket，SSH 进 daemon 机器）
- ~~§11.J Auth 复杂度~~ ✅ 已锁定 2026-04-27（D-16）：J2 简化版（access + refresh，无 token-family）
- ~~§11.H Server 存储~~ ✅ 已锁定 2026-04-27：SQLite
- ~~§11.I 部署形态~~ ✅ 已锁定 2026-04-27：I3 双路径并行支持，I1（同机）默认推荐
- ~~§11.K Connectivity 文档~~ ✅ 已锁定 2026-04-27：3 套 setup guide（Tailscale Funnel / Cloudflare Tunnel / 公网 IP + Caddy）
- ~~§11.L CLI 命令列表~~ ✅ 已锁定 2026-04-27：单二进制 `tether` 多子命令（`server` 也是 subcommand）
- ~~§11.N JSONL → Wire mapper~~ ✅ 已锁定 2026-04-27：参考 happy mapper，Go 精简版 ~300-400 LOC
- ~~§11.O 跨平台~~ ✅ 已锁定 2026-04-27：CLI = Linux + macOS（amd64+arm64），Windows v0.2
- ~~§11.P 分发~~ ✅ 已锁定 2026-04-27：GitHub releases binary + Homebrew tap；Mobile = App Store + Google Play 内测
- ~~§11.Q 测试与 CI~~ ✅ 已锁定 2026-04-27：PoC-1/2/3 转 CI 集成测试 + 单元测试 + Mobile E2E 用 tauri-driver + 跨平台 matrix
- ~~§11.R 可观测性~~ ✅ 已锁定 2026-04-27：JSON log + tether status + tether log + audit log；Prometheus 推迟到 v0.2

- ~~AgentProvider 抽象 (§11.W)~~ ✅ 已锁定 2026-04-27（D-17）：daemon 跟具体 agent CLI 之间用 `AgentProvider` 接口；v0.1 仅实现 `ClaudeCodeProvider`，架构为 codex / opencode / harness 等留路；wire envelope 加 `providerType` 字段，App 按 providerType 分发 per-provider renderer
- ~~v0.2 Phase 1 forward-compat 预留 (§11.X)~~ ✅ 已锁定 2026-04-27（D-18）：基于 `2026-04-27-phase1-dag-protocol.md`，v0.1 做 9 项预留（workspaceRoot / `/hooks/` 前缀 / `/blob/*` 路由 / blob E2E contract / App 渲染 pipeline / `sendUserMessage` API / `<workspace>/.tether/` 自动创建 / `tether-blob-register` stub / `catchup.state-snapshot` kind），共 ~135 LOC，避免破坏性升级

### 2026-04-28 Staff Engineer Challenge Review 后调整

经一轮 challenge review（详见 review prompt + findings），以下决议**调整 / 收紧**：

- **D-13 收窄**：v0.1 = Android only（推迟 iOS 到 v0.1.x），节省 ~1 周 + 避免 App Store 审核延迟 + iOS quinn polyfill 不确定性
- **D-17 reframe**：从"Multi-agent provider 架构"改为"AgentProvider internal seam"——v0.1 不公开多 provider 能力（无 `--agent` flag、无 `tether agents` CLI）
- **D-12 重命名**："forward secrecy" → "server-compromise confidentiality"——磁盘冷攻击 + 活 session 下能解历史，原命名误导
- **D-18 reframe**：明确"路径占位 + 关键 seam，**不承诺 DAG ready**"
- **PoC-2 升级为 HARD GATE**：失败必须降级 raw QUIC HTTP/3，不引入 WS/TCP fallback
- **新增 PoC-2.5**：PTY ring overflow stress（cc 5MB / 50MB 单次输出）
- **§11.C 加强**：QR 含 fingerprint + server fallback SAS；replay window 5min + ID dedup LRU；威胁模型显式化（server 入侵 = metadata leak + replay risk + cipher 永存）
- **§11.E 重整**：v0.1 仅 FCM（D-13 Android only），APNs 推迟到 v0.1.x；alert push 是 baseline，silent push wake 待 PoC-2 之后独立验证
- **§11.E JPush 升级为独立 milestone**（5-10 person-days，非 60 LOC 小补丁）
- **§11.R 加强**：`tether doctor` 自检 + `tether support-bundle` redacted debug 包 + audit log schema 字段化 + cc 兼容 CI gate
- **§6.6 新增**：三类恢复路径（健康 / 可 resume / 需重新配对 / 只能新建 session）
- **§3.3 加强**：envelope replay window + dedup 显式

**v0.1 设计阶段完结**。18 条 D-XX 决议入 §2。**v0.1 timeline 收窄到 10-12 周（Android only）**，iOS 增量 ~1.5-2 周（v0.1.x 独立 milestone）。后续工作 = PoC-2 hard gate / PoC-2.5 ringbuf / PoC-3 watchdog 三个验证。

### 2026-04-27 PoC-1 日志

PoC-1 完整 12 步在 2026-04-27 一天内跑完，11 条事实见 Appendix B。关键发现：

1. **F-01 / F-07** 改写了 §5.3："Submit 是 `\r` 不是 `\n`"，"取消 turn 序列 = `Esc + Ctrl+U + 新 prompt + \r`"
2. **F-02** 加了 trust prompt 自动化方案（§5.3 末尾）
3. **F-04 / F-09** 改写了 §6 + §5.2 daemon 工作分解：JSONL 三类 + 防御要求
4. **F-05 / F-06** 加了 hook 5 类完整覆盖 + payload schema（§5.2）
5. **F-10 / F-11** 是最大的设计简化：发现 cc 自带三层权限模型（plan / safe-cmd / mode+hook+TUI），**daemon 不需要造任何"权限锁"**。这让 §11.D 从"双层 lock 复杂设计"简化成"单层 lock + permission_mode RPC"

代码 ~1900 LOC Go 在 `poc/go-pty-cc/`，每步独立 build tag、可重跑。

后续讨论可以基于这 11 条事实直接进入 §11.A/B/C 拓扑层决策，cc 集成层的不确定性已经全部消除。

---

## Appendix B：PoC-1 验证事实清单（11 条单一引用源）

下面的 F-XX 编号在全文其他位置被引用，是**单一事实源**。

| # | 事实 | 影响章节 |
|---|---|---|
| **F-01** | cc TUI 在 raw mode 下 Submit 用 `\r`（CR），不是 `\n`（LF）。daemon 写 PTY 时换行用 `\r` | §5.2 §5.3 |
| **F-02** | 新目录首次启动 cc 弹"trust prompt"（默认选 "Yes I trust"），daemon 启动后等 ~3s 发 `\r` 接受 / 用 `--add-dir` / `permissions.allow` settings 预登记 | §5.3 |
| **F-03** | `--session-id <uuid>` 工作；JSONL 路径完全可预测；编码规则 `/x/y/z` → `-x-y-z`（`/` 全替换为 `-`） | §5.4 |
| **F-04** | JSONL 记录三类：**EVENT**（type=user/assistant，有 UUID）/ **HOOK**（type=attachment，含 hookEvent + hookName）/ **STATE**（permission-mode / last-prompt / file-history-snapshot，无 UUID） | §5.2 §5.6 |
| **F-05** | Hook payload schema 固定字段：`session_id` `transcript_path` `cwd` `permission_mode` `hook_event_name` `tool_name` `tool_input` `tool_use_id`；PreToolUse 响应用 `hookSpecificOutput.permissionDecision`（`allow`/`deny`/`ask`） | §5.2 |
| **F-06** | 5 类 hook 形成完整事件流：**SessionStart / UserPromptSubmit / PreToolUse / PostToolUse / Stop**。Hook 是 daemon 的第二条独立可见通道（vs JSONL）：实时 + 结构化 + 可拦截 | §5.2 |
| **F-07** | 取消 in-flight turn = `\x1b` (Esc) + 等 500-800ms + `\x15` (Ctrl+U) + 等 200-500ms + 新 prompt + `\r`。**禁用 SIGINT**（cc 进入僵尸态） | §5.3 |
| **F-08** | 多 cc session 100% 隔离（PID/PTY/JSONL 全独立）；单 hook server 服务全部 session 没问题，靠 payload `session_id` 路由 | §3 §5.2 §5.5 §9 |
| **F-09** | cc JSONL 写入实测原子（1ms × 45s × 10.6KB record，0 torn write），但 daemon 仍须做"末尾非 `\n` 视为未完整"的防御 | §5.6 |
| **F-10** | cc 三层权限：(1) `plan` 模式 model-level 拒绝 tool；(2) 内置 safe-cmd allowlist（echo/ls/cat 等无副作用命令）自动放行不走 hook；(3) `default` 模式下非 safe 命令走 TUI 对话框，hook 是 pre-answer 通道。**daemon 无需造权限锁**——用 cc 的 mode 切换即可 | §5.5 §11.D |
| **F-11** | `--permission-mode plan` 真 fail-closed（含 echo），且**不调 hook**——是模型层 system-prompt 自我约束。"agent 只读"功能开箱即用 | §5.5 |

每条事实都有对应的 `poc/go-pty-cc/step*.go` 复现脚本。如果未来 cc 升级行为变化，重跑这些脚本可以快速发现差异。

---

## Appendix C：PoC-2 验证事实清单（webtransport-go v0.10）

PoC-2 进行中（2026-04-28 截至 step 1+2+5 PASS）。下面 P-XX 是 webtransport-go v0.10 的**实测行为事实**，每条都有 `poc/go-quic-wt/step*.go` 复现。

### C.1 库行为事实（4 条）

| # | 事实 | 触发场景 |
|---|---|---|
| **P-01** | `webtransport.Server.ListenAndServe` **不会**自动配置 H3 setting `enableWebtransport`。必须**显式调用 `webtransport.ConfigureHTTP3Server(h3)`** —— 这会设 `AdditionalSettings[settingsEnableWebtransport]=1` + `EnableDatagrams=true` + `ConnContext`。错过时客户端报 "server didn't enable WebTransport" | server 端初始化 |
| **P-02** | `EnableStreamResetPartialDelivery: true` 是 webtransport-go v0.10 **硬性要求**，server 和 client 的 `quic.Config` 都必须显式开。错过时报 "stream reset partial delivery required, enable it via QUICConfig.EnableStreamResetPartialDelivery" | dial 时立即报错 |
| **P-03** | Datagram 必须**两层都开**：`http3.Server.EnableDatagrams = true`（RFC 9297 HTTP/3 datagram 协商）+ `quic.Config.EnableDatagrams = true`（QUIC datagram extension）。仅开一层时报 "server didn't enable HTTP/3 datagram support" | dial 时立即报错 |
| **P-04** | 客户端 `tls.Config.NextProtos` 必含 `"h3"`，否则 ALPN 协商失败 → handshake error | TLS handshake |
| **P-05** | 浏览器 WebTransport 通过 `serverCertificateHashes` pin 自签 cert 时，**Chrome 仍执行 RFC 6125 hostname verification**——cert SAN 必须包含 dial host（IP 或 DNS）。仅写 127.0.0.1 不够，必须把 server 实际外部 IP / 域名加进 SAN。否则 dial 直接 `WebTransportError: Opening handshake failed`（quic_error=0，cert verify 层挡）| step 1 server cert generation |
| **P-06** | **本地 HTTP/SOCKS proxy（Clash / Surge / V2Ray 等）会拦截 WT 流量并 fail**——proxy 用 HTTP CONNECT 隧道，不支持 UDP/QUIC，返回 `ERR_TUNNEL_CONNECTION_FAILED`（net_error=-111, quic_error=0）。Chrome netlog 里 `PROXY_RESOLUTION_SERVICE_RESOLVED_PROXY_LIST` 显示 `PROXY 127.0.0.1:7897`（或类似端口）即此问题。**修复：在 proxy 工具的 rules 里加 `IP-CIDR,<server-ip>/32,DIRECT`，或全局切到直连模式**。tether 部署文档（§11.K）必须写清这条 | client 端配置 |

### C.2 Step 5 多 stream + datagram 实测数据（localhost）

PASS 标准：4 streams 真正并行不互堵 + datagram 跟 stream 并存 + 大 blob 不引发 HOL blocking。

```
=== Per-stream stats（5MB blob 上行同期）===
  control   100 msg, 32B each   p50=272µs   p99=5.5ms
  events    100 msg, 32B each   p50=269µs   p99=5.8ms
  a-bytes   1× 5MB blob         throughput=1045.7 Mbps（40ms 完成）
  catchup   100 msg, 32B each   p50=281µs   p99=5.7ms

=== Datagram heartbeat（5Hz × 6s）===
  sent=30  recv=30  loss=0%

=== Wall === 6.5s（受 dgram 6s 时长限制）
```

**关键 invariant**：a-bytes stream 5MB 大数据上行**期间**，control / events stream 的小消息 RTT 仍维持 < 6ms p99——证明 webtransport-go 的 stream 多路复用在 lib 层面真的并行，不串行化。

### C.3 Step 6 长连接 30 分钟实测数据（localhost，2026-04-28）

PASS 标准：webtransport-go v0.10 client mode 在 30 分钟连续运行下无 panic / 无 reconnect / 无 leak / 心跳稳定。

```
=== Final stats（30:00 wall, localhost）===
  control  sent=360  failed=1*  p50=430µs  p99=900µs  max=1.04ms
  events   sent= 60  failed=1*  p50=439µs  p99=607µs  max=1.03ms
  datagram sent=1800 recv=1799  loss=0.06%
  heap     start=0MB  end=1MB   growth=1MB

=== Periodic snapshots（每 5 min）===
  +5min   ctrl 59/0   evt 10/0   dgram 299/299 100%   heap=0MB
  +10min  ctrl 120/0  evt 20/0   dgram 600/599 99.8%  heap=1MB
  +15min  ctrl 180/0  evt 30/0   dgram 899/899 100%   heap=1MB
  +20min  ctrl 239/0  evt 40/0   dgram 1200/1199 99.9% heap=1MB
  +25min  ctrl 300/0  evt 50/0   dgram 1500/1499 99.9% heap=1MB
  +30min  ctrl 360/0  evt 60/0   dgram 1799/1799 100%  heap=1MB

* 两个 failed=1 都是 30min 到点 ctx canceled 时 stream read 超时
  （正常关闭路径），不是连接 / 协议错误。
```

**关键 invariant 全部达标**：
- **0 stream failure**（连接持续 30 分钟，5s/30s 周期 ping 全部 echo 成功）
- **0 reconnect**（webtransport-go client mode 一根 session 撑满 30 分钟）
- **heap growth 1MB**（远低于 50MB 阈值，无 leak）
- **datagram 99.9%**（loss 0.06%，远好于 5% 容忍线；唯一一个 dgram 在 +5min 那个 5min 窗口里）
- **RTT 平稳**：control / events 30 分钟 p99 都在 1ms 内浮动，无衰减趋势

**§11.T 三大风险点**（webtransport-go client mode / 长连接稳定性 / heartbeat）**全部消除**。**保守判断**：webtransport-go v0.10 client mode 在生产中可作为 v0.1 CLI transport 直接使用，**没有 fallback 到 raw QUIC 的迹象**。

**caveats**：
- localhost 测试无网络抖动 / 无 4G NAT 行为；step 4 真机测试是补全这一面的最后一步
- 30 分钟 ≠ 几小时 / 几天的真实 dogfood 周期；实施期 dogfood 自然会暴露更长周期的稳定性问题（如有）
- 模拟丢包（`tc qdisc add ... netem loss N%`）未做——纯 v0.1.x 增量 PoC，不阻塞

### C.4 Step 3 Browser ↔ Go server 互通实测（2026-04-28）

PASS 标准：desktop Chrome 通过公网 IP 用 `serverCertificateHashes` 连 GCP VM 上的 webtransport-go server，5 项 ✓。

**测试环境**：
- Server: GCP VM (asia-northeast1-c) external IP `34.104.167.170`
- Server cert SAN: `127.0.0.1, ::1, 34.104.167.170, 10.146.0.11, localhost, tether-poc.local`
- 桌面：macOS Chrome，HTML via step3/HTTP/8003 + Chrome flag `unsafely-treat-insecure-origin-as-secure` 启用
- 移动：**Android Chrome**，HTML via Cloudflare quick tunnel HTTPS（secure context 自动满足，无需 flag）

**结果**：
```
DER hash:  6d8dae1ba9ebcefe7b0b8816dcb59ec05ad272636bf1c3b65f73df59084d0bb3
SPKI hash: d2fcb20d13ee095d50e8b77b272215e3b985e595294572256c1187838ff48930
dialing https://34.104.167.170:4433/wt with 2 hash(es)
✓ session ready
✓ bidi echo
✓ datagram echo
✓ multi-stream: ["ctrl-msg","evt-msg-12345","agent-bytes-payload-x"]
✓ closed cleanly
```

**根因调查路径（值得记录）**：

1. 第一次试 → `Opening handshake failed`，看似 cert/TLS 问题
2. 写 `cert_dump.go` 比对 Chrome `serverCertificateHashes` 全部 8 项要求——cert 100% 合规（ECDSA P-256 / SHA-256 / 169h validity / leaf / CN / SAN 含外部 IP / 双向 KeyUsage），排除 cert 问题
3. 加 SPKI hash 双发——仍 fail
4. Chrome `chrome://net-export/` 抓 netlog → 看到关键证据：
   ```
   PROXY_RESOLUTION_SERVICE_RESOLVED_PROXY_LIST: "PROXY 127.0.0.1:7897"
   QUIC_SESSION_WEBTRANSPORT_CLIENT_STATE_CHANGED:
     net_error: -111  ERR_TUNNEL_CONNECTION_FAILED
     quic_error: 0    ← QUIC 层完全 OK
   ```
5. → 用户机器跑 Clash Verge 系统代理，HTTP CONNECT 隧道不支持 UDP/QUIC，proxy 直接拒
6. 用户在 Clash 配置 `prepend-rules: IP-CIDR,34.104.167.170/32,DIRECT` 后立即通过

**Lesson**：`serverCertificateHashes` 的 W3C spec 要求和 Chrome 实际行为之间有细节差异（P-05 SAN check）；同时**用户端 system proxy 是不可忽略的真实部署阻塞点**（P-06）。这两条都进了 Appendix C 的 P-XX 事实清单，部署文档（§11.K）必须显式提示用户配置 proxy bypass。

**Android 路径补充结果**（2026-04-28，同日跟 desktop 一起验）：
- Cloudflare quick tunnel（`cloudflared tunnel --url http://localhost:8003`）给 HTML 一个公网 HTTPS URL，secure context 自动满足，**无需 Android Chrome flag 配置**
- HTML 加载后，"WT host:port" 字段必须**手动改成 `<server-public-ip>:4433`**（默认从 `window.location.hostname` 派生，会错填成 tunnel 域名+4433）
- WT 实际流量直连 GCP VM:4433（不走 tunnel），所以 GCP firewall UDP/4433 + Clash bypass 仍是前提
- 5 项全 ✓——**直接验证 D-13 Tauri Android only 决议的核心可行性**（Android Chromium WT 跟 Go server 互通无差异）



### C.5 §11.T 风险点状态映射

文档原 §11.T 写："风险点 → PoC-2 必验：webtransport-go 的 client mode 在长连接 + 心跳 + connection migration 下的稳定性。"

| 风险点 | PoC-2 验证状态 |
|---|---|
| client mode 基础稳定性 | ✅ step 2 + step 5（短连接 + 多通道并发）|
| 长连接 + 心跳稳定性 | ✅ **step 6**（30 分钟实测：0 panic / 0 reconnect / 1MB heap / 99.9% dgram）|
| Browser ↔ Go server 互通 | ✅ **step 3**（公网 GCP VM + Chrome 全 5 项 ✓ over `serverCertificateHashes`）|
| connection migration | ⏳ step 4（真机 + 网络切换） |

**结论**（截至 2026-04-28）：**§11.T 核心风险已完全消除**——**webtransport-go v0.10 不需要降级到 raw QUIC**。step 4 真机 migration 是补全网络抖动场景的最后一步，但**不阻塞实现启动**。

### C.6 复现命令

```bash
cd poc/go-quic-wt
go build -tags step1 -o bin-step1 .
go build -tags step2 -o bin-step2 .
go build -tags step3 -o bin-step3 .
go build -tags step5 -o bin-step5 .
go build -tags step6 -o bin-step6 .

# 终端 A
./bin-step1
# 终端 B（任选一）
./bin-step2                          # 基础 echo（step 1+2 验证）
./bin-step5                          # 多 stream 并行 + datagram heartbeat
./bin-step6                          # 30 分钟长连接（建议后台跑：./bin-step6 > /tmp/step6.log &）
STEP3_PORT=8003 ./bin-step3          # 静态 HTTP server 给浏览器测试用
```

每个 step 独立 build tag、可重跑。如果 webtransport-go 后续版本行为变化，重跑这些脚本可以快速发现差异。

---

**文档结束。**
