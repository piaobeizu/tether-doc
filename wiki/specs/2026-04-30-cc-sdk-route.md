# Spec: cc integration 从 PTY 改 SDK 路线（NDJSON over stdio）

| 字段 | 值 |
|---|---|
| 状态 | draft |
| 起草日期 | 2026-04-30 |
| 作者 | wxk |
| 关联 issue | piaobeizu/tether#13 |
| 父 epic | #3 Daemon goroutine + cc integration（v0.1 foundation）|
| 来源材料 | `<workspace-root>/SESSION-HANDOFF-2026-04-30-sdk-migration.md`（commit M5 时迁入 `tether-doc/wiki/specs/`） |
| 论证记录 | 7 站可行性论证（A/B/C/D/E/F/G）+ cwd-gap 实测 |

---

## 1. Problem

Epic #3 原计划：tether 用 PTY 包 `claude` 子进程，截 ANSI 文本流，模拟键盘输入。

2026-04-30 调研 [happy-cli](https://github.com/slopus/happy-cli) 的实地代码后确认：Anthropic 官方 CLI 已支持 `--output-format stream-json --input-format stream-json --verbose`，提供完整的**结构化 NDJSON 接口**（数据平面 + 控制协议 + session 持久化 + resume），与 PTY/ANSI 路线相比：

- **零 ANSI 解析**——结构化事件直接 `json.Unmarshal`
- **零 PTY 依赖**——纯 OS pipe，跨平台一致
- **零 Node 嵌入**——`claude` 已是原生 ELF 二进制（v2.1.123 实测），Go 直接 spawn
- **一等公民接口**——Anthropic 自己的 Python/TypeScript Agent SDK 都建立在同一组 flag 上，长期投资接口

**决定（用户 ACK 2026-04-30）**：tether 弃 PTY 路线，改走 stream-json 路线。本 spec 把这一决定 + 7 站可行性论证产出的契约 + 实测 cwd-gap 落成正式工程契约。

---

## 2. Goals

1. tether daemon spawn `claude` ELF binary，以 NDJSON over stdio 通信，完整覆盖：数据平面（user/assistant/result）、控制平面（control_request/control_response）、session 生命周期。
2. session 持久化 100% 委托给 CC 自己（写 `~/.claude/projects/<encoded-cwd>/<sid>.jsonl`），tether 仅读不写。
3. session 恢复以 `claude --resume <sid>` 为唯一原语，封装为 `Session.Recover()`，被四个 caller 共用：plugin reload / pod restart / daemon crash / WT reconnect。
4. v0.1 部署形态：k8s + JuiceFS PV + 一用户一容器，tether daemon 与 cc 共驻容器内。
5. 1–2 小时 spike 在 `internal/backend/claude/` 验证：完整对话、至少一次工具授权、resume 续接、子进程干净退出。
6. 产出后续 sub-ticket 拆分清单。

---

## 3. Non-goals

分两类：**完全推迟到 v0.2+** 的、**v0.1 milestone 内但不在本 ticket** 的。

### 3.1 推迟到 v0.2+（v0.1 milestone 不做）

| 项 | 触发条件 |
|---|---|
| Backend interface 抽象（CC/Codex/OpenCode 共享接口） | 真实第二个 backend 落地时（由 diff 倒逼抽象点）|
| 跨设备 session 直接迁移（笔记本**裸 cc resume** 容器 session） | 用户群里出现"不想装 tether 客户端，要裸 cc 续接"的明确反馈时——加同步层（笔记本挂 JuiceFS / `tether export` 子命令） |
| Container provisioning + cold-start optimization | SaaS 化上线（独立 epic 量级）|
| `/login` `/init` `/clear` `/config` `/mcp` 等 CC 交互模式 slash 命令的 tether 等价实现 | 各自独立 ticket，按用户投诉密度排（注：这些只在 CC 交互模式可用；tether 走 `--print` 非交互模式不能直接转发，所以"等价实现"是 tether 自己重写一遍。其中 `/clear` 在 tether 里语义最轻——等价于"开新 session"，spawn 新 claude 不带 `--resume` 即可；`/login` `/mcp` 量级最重）|

### 3.2 v0.1 milestone 内，但不在本 ticket（#13）

| 项 | 归属 |
|---|---|
| Tauri native 渲染层（markdown 流 / tool_use 卡片 / diff 渲染 / session 列表 UI） | Epic #6 App UI 全 surface |
| 自写 WT Tauri command | Epic #7 包 web-transport-quinn |
| E2E 加密层（用户设备 ↔ tether daemon） | Epic #5 X25519 + XChaCha20-Poly1305 + HKDF |

**本 ticket 与上述三个 epic 的协作面**：

- Epic #6 / #7：本 spec 产出 "daemon 给 UI 推什么消息流"的协议定义（即 tether 自己的 canonical event schema），UI 怎么渲染、怎么经 WT 传输由 #6 / #7 各自决定。
- Epic #5：本 spec 不在 daemon ↔ cc 这一段做加密（OS pipe 是内核内存，root 用户进程间，不构成 attack surface）。加密层在用户设备 ↔ tether daemon 这一段，由 Epic #5 独立设计。

### 3.3 判定标准

§3.1 各项过三问：(1) v0.1 主路径必经？(2) 工程量级 ticket 还是 epic？(3) 不做卡 v0.1 落地？三问全 no → 推迟。

§3.2 各项是 v0.1 必做，但**职责边界清晰可拆**——本 ticket 单独完成可以独立达到 DoD（spec + spike），不依赖 #5 / #6 / #7 同步进展。

---

## 4. Approach

### 4.1 数据流总览

```
   用户设备（手机 / 笔记本 / Tauri app）
              │
              │ QUIC + WebTransport（tether 自有协议）
              ▼
   ┌──────────────────────────────────────┐
   │  k8s pod（用户专属，PV 来自 JuiceFS）│
   │                                      │
   │  ┌────────────────────────────┐      │
   │  │  tether daemon (Go)        │      │
   │  │   ├ WT 连接管理            │      │
   │  │   ├ Session.Recover() 引擎 │      │
   │  │   └ claude wrapper         │      │
   │  │       │                    │      │
   │  │       │ exec.Command       │      │
   │  │       │ stdio = 3 pipes    │      │
   │  │       ▼                    │      │
   │  │  claude (ELF binary)       │      │
   │  │   --print                  │      │
   │  │   --output-format stream-json     │
   │  │   --input-format stream-json      │
   │  │   --verbose                       │
   │  │   --permission-prompt-tool stdio  │
   │  │   [--resume <sid>]                │
   │  │       │                           │
   │  │       │ NDJSON over stdin/stdout  │
   │  │       │   ├ system/init           │
   │  │       │   ├ stream_event          │
   │  │       │   ├ assistant / user      │
   │  │       │   ├ result                │
   │  │       │   └ control_request /     │
   │  │       │     control_response      │
   │  │       │                           │
   │  │       │ jsonl 写盘                │
   │  │       ▼                           │
   │  │  ~/.claude/projects/              │
   │  │    <encoded-cwd>/<sid>.jsonl      │
   │  │   ↑                               │
   │  │   └─ JuiceFS 挂载                 │
   │  └────────────────────────────┘      │
   └──────────────────────────────────────┘
              │
              │ HTTPS
              ▼
          Anthropic API
```

### 4.2 spawn 调用（Go）

```go
cmd := exec.Command("claude",
    "--print",
    "--output-format", "stream-json",
    "--input-format",  "stream-json",
    "--verbose",
    "--permission-prompt-tool", "stdio",
    // resume 模式额外加：
    "--resume", sessionID,
)
cmd.Dir = projectCwd  // tether 的 cwd 策略决定（见 §6.E.5）
stdin,  _ := cmd.StdinPipe()
stdout, _ := cmd.StdoutPipe()
stderr, _ := cmd.StderrPipe()
cmd.Start()
```

三个 pipe，纯字节流。无 PTY，无 fd 3。

### 4.2.1 进程生命周期（重要）

`--print` 的两种子模式：

| 子模式 | 行为 | tether 用哪个 |
|---|---|---|
| `-p "single prompt"`（不加 `--input-format stream-json`） | 收一个 prompt → 跑一轮 → 发 `result` → **进程退出** | no |
| `-p ... --input-format stream-json` | stdin 持续读 NDJSON `user` envelope → 每收一条跑一轮 → `result` → **继续等下一条**，直到 stdin 关闭才退出 | yes |

tether 必须用第二种（`--input-format stream-json`）——`claude` 进程**与 tether daemon 共活**，多轮对话**复用同一进程**。否则每轮 spawn 一次的 startup + cache_creation 开销根本撑不住（实测 ~$0.10/spawn，Haiku）。

`Session.Recover()` 触发时，杀的是这个 long-running 进程，再 spawn 新的并带 `--resume <sid>` 续接。**不是每轮 spawn**。

### 4.2.2 slash 命令在 `--print` 模式都不可用

CC 的交互模式 slash 命令（`/init`, `/login`, `/clear`, `/config`, `/mcp`, `/resume` 等）**只在不带 `-p` 的交互 REPL 里可用**——`--print` 任何子模式都没有 slash 命令通道。tether 想给用户提供等价功能必须自己重写（见 §3.1）。

### 4.3 session_id 发现

不走 happy-cli 的 SessionStart-hook 中转。直接 parse 第一条 `system/init` 事件：

```go
{"type":"system","subtype":"init","session_id":"c840b5fb-...",...}
```

第一条事件就拿到 sid，记到 `Session` 结构里。

### 4.4 Session.Recover() 统一抽象

```go
type Session struct {
    SessionID string
    ProjectCwd string
    // ...
}

// Recover 是所有"断开重连"场景的唯一入口。
// 调用方：plugin reload / pod restart / daemon crash 重启 / WT 重连。
// 前置约束：必须满足 §6.C.1–C.5 五条护栏。
func (s *Session) Recover(ctx context.Context) error {
    // 1. 杀掉残留子进程（如果有）
    // 2. spawn claude --resume s.SessionID 在 s.ProjectCwd
    // 3. 等 system/init，校验 session_id 一致
    // 4. 恢复 stream 推送给 WT 客户端
}
```

### 4.5 控制协议（双向 control_request / control_response）

参考 happy-cli `src/claude/sdk/query.ts:174` 的握手实现，逻辑约 80 行 TS，直译 Go 约 100–150 行。

**inbound（cc → tether）**：`control_request` 携 `subtype: can_use_tool` —— tether 调用户回调（或自动策略）决定 allow/deny，回写 `control_response`。

**outbound（tether → cc）**：`control_request` 携 `subtype: interrupt` 等 —— tether 主动取消 cc 当前推理。

---

## 5. Trade-offs

| 维度 | 收益 | 代价 | 缓解 |
|---|---|---|---|
| stream-json vs PTY | 结构化、零 ANSI、稳 | 协议未全文档化，控制平面靠 RE | A.2 versioned parser + happy/companion 两家独立 RE 互证 |
| 复用 CC jsonl | 免重写存储、免重做 resume 机制 | 格式是 CC 内部 schema，tether 跟字段绑定 | E.2 只取稳定子集 |
| kill+resume 作为唯一恢复原语 | 简单、无双执行风险 | 任务意图可能丢失（见 §7.1）、单次 ~$0.05–0.15 | C.5 UX 提示；"重要任务 reload 前用户确认" 留 v0.2 |
| cwd-bucketing | CC 已实现，零工程量 | 用户裸 cc resume 找不到 session（实测 gap） | E.5 显式 cwd 策略 + 文档化 |
| 一容器一用户 | 无多租户隔离层、无并发问题 | per-user pod 资源开销、cold-start 不优化 | v0.1 你自用，不优化；上线时单独 epic |

---

## 6. Behavior contract（24 条）

### A. 接口层

- **A.1** **不依赖 fd 3 副信道**。CC 在 fd 3 上发的 `fetch-start/fetch-end` 是未文档化的诊断 fd，可能任何版本无预警移除。如需 thinking 状态指示，从 in-band `stream_event`（首个 `text_delta` 之前的状态）推断。
- **A.2** **数据 + 控制双平面 versioned parser + unknown-type/subtype graceful-degrade**。`control_request` 协议靠 happy / The-Vibe-Company/companion 两家独立 RE 互证，未官方文档化。parser 必须：
  - 控制平面：按 `subtype` 分发，遇到未知 subtype 写日志并 ack 一个空 success response，**不 abort session**。
  - 数据平面：按 `type` 分发，遇到未知 `type` 同样 log + skip + 不 abort（防御 CC 跨版本新增事件类型）。
- **A.3** **`Send()` 遇 EPIPE / 子进程已死必须返回 typed `ErrSubprocessGone`**。理由：用户 send 一条消息后 caller 必须能区分"消息发出去了，等响应"vs"子进程已死，根本没收到"。这两种 caller 的处理逻辑完全不同（前者等 result，后者触发 Recover）。
- **A.4** **`system/init` wait 必须有硬超时（默认 10s）**，超时返回 `ErrInitTimeout`。理由：bad `--resume <sid>`（例如笔记本 cwd 不一致 / sid 已被 GC）会导致 cc 进程启动后**永远不发 system/init**，无超时则 daemon 永挂。
- **A.5** **stderr 必须持续 drain 到 ring buffer**（不丢但不阻塞）。理由：cc binary 把 stderr 当 debug 通道，pipe 默认 64KB 满了会**阻塞子进程**写入 → 看似进程在跑实际卡死。drain 到 ring buffer，进程退出时一次性 surface 给上层日志。

### C. 恢复语义

- **C.1** **safe-to-reload gate**：`Session.Recover()` 仅在 `result` 事件之后、下一个 user message 之前可触发。tether daemon 维护 `RecoverGate` 状态机。
- **C.2** **tool-pair 完成前禁止 reload**。已发 `tool_use` 但 `tool_result` 未送回时，gate 关闭。
- **C.3** **mid-text-stream 禁止 reload**。`stream_event` 中途的 `text_delta` 未到 `message_stop` 时，gate 关闭。
- **C.4** **如 tether 自装 hooks，必须 idempotent**。v0.1 spike **不**装任何 tether-owned hook（保持纯净）；后续 ticket 装时此约束生效。理由：`SessionStart:resume` hook 在每次 recover 都重新触发，副作用 hook 重发会导致用户感知错误。
- **C.5** **reload 成本预算 ~$0.05–0.15/次**（Haiku 实测；Sonnet/Opus 待 spike 实测）。UX 必须给用户"正在重载会话…"反馈，不能默默扣钱。**注：UX 反馈半边依赖 Epic #6 UI**，本 ticket 只在 daemon 端 emit `RecoveryStarted` / `RecoveryCompleted` 事件，UI 渲染由 #6 实现。
- **C.6** **`Recover()` 必须先关闭 pending control_request map**。SIGKILL 子进程前，遍历 pending responses map，每个未完成 request 用 sentinel `ErrRecoverInterrupted` 关闭。理由：不清理 → goroutine forever-block 在 channel 等已经死掉的子进程的回应 → 内存泄漏 + 上层逻辑挂。

### D. 部署 & 持久化

- **D.1** **JuiceFS 持久化路径清单**：`~/.claude/projects/`、`~/.claude/plugins/`、`~/.claude/auth/`、`~/.claude/settings.json`。其它 `~/.claude/*` 路径若 spike 阶段发现新增依赖，加入此清单。
- **D.2** **Session.Recover() 是统一恢复原语**。plugin-reload、pod-restart、daemon-crash、WT-reconnect 全部走同一入口，共用 C.1–C.5 护栏。
- **D.3** **Container provisioning + cold-start optimization 显式 v0.1 out-of-scope**（见 §3）。

### E. Session 存储 & 生命周期

- **E.1** **CC 拥有 jsonl 写入权，tether 只读不写**。任何"tether 主动修改 jsonl" 的实现都是 spec 违反。
- **E.2** **tether 维护自己的对话投影模型**供 UI 渲染，仅依赖 jsonl 稳定子集：`type` / `role` / `content` / `sessionId` / `parentUuid` / `timestamp`。`gitBranch` / `cwd` / `entrypoint` / `isSidechain` / `queue-operation` / `attachment` 等 CC 内部字段 **不进** tether 模型。**v0.1 spike 不实现投影层**（UI 渲染归 Epic #6）；本契约 forward-looking 适用于 v0.1 follow-up #6 + Epic #6 实施时。
- **E.3** **GC 失败/auth-error stub session**。扫描 `~/.claude/projects/<dir>/*.jsonl` 时过滤 `size < 5KB AND (type=result with is_error=true OR error="authentication_failed")` 的 stub session，不展示给用户。**v0.1 spike 不实现**——deferred to follow-up sub-ticket #5（GC sweeper）。本契约定义过滤规则；后台 sweeper goroutine 由 #5 实施。
- **E.4** **跨设备直接 `cc --resume` v0.1 不支持**。预设产品形态 = 所有设备装 tether 客户端，session 永远走 daemon。
- **E.5** **tether 必须显式定义 cwd 策略并文档化**。理由：cc `--resume` 是 cwd-scoped，用户裸 SSH 进容器后 `claude --resume <sid>` 在错误 cwd 下会得到 `No conversation found`（实测确认）。v0.1 推荐策略：
  - **(a)** 永远用 fixed cwd（`$HOME`）spawn——所有 session 进 `~/.claude/projects/-root/`，简单
  - **(c)** 提供 `tether resume <sid>` wrapper——自动 cd 到正确 cwd 再 spawn，屏蔽 cwd 概念

  v0.1 spike 阶段二选一定稿（开放问题 §10.4）。(b)(d) 留 v0.2+。

### F. 实现路径

- **F.1** **不嵌 Node**。Go 直接 `exec.Command("claude", ...)` spawn ELF binary。理由：
  - claude v2.1.123 已是原生 ELF（实测 `/proc/<pid>/maps`），不再是 Node bundle
  - 嵌 npm 包 `@anthropic-ai/claude-agent-sdk` 内部仍 spawn 同一个 ELF（链路：Go → Node → ELF）；多一层 Node runtime 开销，无收益
  - SDK 若自实现 agent loop（不调 CC）则丢失 hooks/plugins/MCP/skills，违背"做 CC 客户端"目标
- **F.2** **`claude` binary 缺失必须返回 typed `ErrBinaryNotFound`**。`Spawn()` 在 `exec.LookPath` 失败或 `cmd.Start` 报 `ENOENT` 时返回明确错误（含搜索路径），而非泛型 `*os.PathError`。理由：daemon 启动时 binary 缺失是头号 deploy 故障；明确错误让 ops 一眼定位。

### G. 后端抽象

- **G.1** **删 handoff 的"共用 spawn-NDJSON 抽象"**。CC 与 Codex 在 transport 之上的所有层（事件 schema / 控制协议 / 会话 ID 概念 / 恢复 API）都不同——抽象点定错了。
- **G.2** **v0.1 硬编码 Claude，不做 Backend interface**。YAGNI：抽象单一实现 = 把 CC 假设固化成"通用接口"，未来真要加 Codex 时被错抽象反咬。
- **G.3** **未来抽象的正确层级**（v0.2+ 加 Codex 时启动）：`SendUserMessage(text)` / `StreamEvents() chan canonical.Event` / `ApproveTool(callID, allow)` / `Resume(sessionIDOpaque)`——每 backend 各自的 wire-adapter 把 NDJSON 翻译为 tether canonical event。

---

## 7. Edge cases

### 7.1 mid-tool-call kill（已实测）
- 现象：assistant 发 `tool_use Bash(...)` 后、`tool_result` 之前 SIGKILL → resume → API **不报错**，模型**不自动重跑**该 tool，conversation 历史保留 tool_use（无 matching tool_result）。
- 行为：模型把未完成的 tool 视作"无结果"，等待新的 user message。任务意图**丢失**（用户感知："我让它做的事它好像忘了"）。
- 处理：tether 在 `Session.Recover()` 后给用户 UI 提示："会话已恢复，刚才的工具调用未完成，请重新描述意图。"

### 7.2 多 tether 进程并发
- 现象：理论上不应发生（单容器单 daemon）；但 daemon 崩溃重启的瞬间可能短暂存在两个进程。
- 处理：tether daemon 启动时检测 PID lockfile（`/var/run/tether.pid` 或 `~/.tether/daemon.pid`），冲突时拒绝启动。

### 7.3 失败 jsonl 累积
- 现象：auth-error 等失败 session 会在 `~/.claude/projects/<dir>/` 留 ~2KB stub jsonl。
- 处理：E.3 GC 过滤 + 后台周期清扫（独立 sub-ticket）。

### 7.4 cwd 不一致 → 用户裸 cc resume 失败
- 现象（已实测）：tether 在 cwd `A` 起 session，用户 SSH 进容器后 cd 到 `B`（如 `$HOME`）跑 `claude --resume <sid>`，CC 报 `No conversation found`。
- 处理：E.5 显式 cwd 策略；如果选 (c) wrapper 路线，文档明确说"裸 cc resume 不被官方支持，请用 `tether resume <sid>`"。

### 7.5 hooks 在 resume 时重发
- 现象：`SessionStart:resume` hook 每次 recover 都会触发。如果 hook 有副作用（发通知、写 log），重复执行。
- 处理：C.4 idempotency 是 tether 自装 hook 的契约；用户已装的 hook 由用户自己负责（不在 spec 担保范围）。

### 7.6 子进程 mid-`Send()` 崩溃 / EPIPE
- 现象：tether 在 stdin 写 user envelope 时，cc 子进程恰好 panic / OOM-kill。`os.Write` 返回 `EPIPE`。
- 处理（A.3）：`Send()` 立即返回 `ErrSubprocessGone`。caller 触发 `Session.Recover()`。**不重发该 user message**——交给上层 UI 决策（让用户重新输入更安全）。

### 7.7 `system/init` 在 resume 时可能发两次
- 现象（实测见过）：resume 模式下，stream 里出现两个 `system/init`——一个用临时 session_id（hook 阶段），一个用真 session_id（被恢复的）。
- 处理：parser 接受多次 init。Session 字段记录最新 session_id；如果与 `Recover(opts.SessionID)` 不一致，log warning 但不报错（CC 内部行为，不可控）。

---

## 8. Testing strategy（spike）

`internal/backend/claude/` 最小实现，包含：

| 文件 | 职责 |
|---|---|
| `spawn.go` | 构造 `exec.Command`、配置 stdio pipe、启动子进程 |
| `message.go` | NDJSON envelope Go struct（versioned；`unknown_subtype` 字段透传） |
| `parser.go` | `bufio.Scanner` 按行解析，按 `type` 分发 |
| `control.go` | `control_request` / `control_response` 双向握手 |
| `session.go` | `Session` 结构，`Recover()` 实现，sid discovery |

### 必跑 scenarios

1. **完整对话**：发 prompt → 收 system/init → stream_event → assistant text → result。
2. **工具授权**：触发 `Bash` tool → 收 control_request can_use_tool → 回 allow → tool_result → assistant 再次 text → result。
3. **拒绝工具**：同上但回 deny → assistant 处理 deny。
4. **mid-text abort**：用 outbound `control_request interrupt` 中断 → 收 result 含 abort 标记。
4b. **mid-text recover 被拒**：在 `stream_event text_delta` 中途主动调 `Recover()` → 必须返回 `ErrUnsafeToReload`（验 C.3 gate）。
5. **resume 续接**：杀子进程在 result 后 → spawn `--resume <sid>` → 验 session_id 一致 → 发新 user message → 收正常 result。
6. **mid-tool-kill + resume**：杀子进程在 tool_use 已发但 tool_result 未到 → resume → 验 §7.1 行为。
7. **hook idempotency**：装一个简单 hook 计数文件 → resume 三次 → 验文件按 idempotent 行为递增（或保持）。

### spike 阶段必补的实测验证（断言但未测）

- **B'**：拉 CC v2.0.x 和 v2.1.x 各跑 scenario 1 一次，diff 两次的 NDJSON 流，确认 envelope schema 兼容。
- **C'**：用 Sonnet 4.6 / Opus 4.7 各跑一次完整长对话（~50K tokens）+ 一次 resume，记录 cost。
- **C''**：在 k8s test cluster 起 pod → 跑 scenario 1 中途 → SIGTERM pod → 等 reschedule → cc startup 是否能 read jsonl from JuiceFS（PV unmount/remount 时序验证）。

---

## 9. Rollout

### v0.1（本 ticket #13 的 DoD）

1. 本 spec 走完 review，迁入 `tether-doc/wiki/specs/2026-04-30-cc-sdk-route.md`（commit M5 promotion）
2. spike 通过 §8 全部 7 个 scenarios + 3 条断言验证（B' / C' / C''）
3. 后续 sub-ticket 拆分清单产出，至少包含：
   - cwd 策略实现（E.5 二选一）
   - Session.Recover() 完整实现（C.1–C.5 状态机）
   - hook idempotency 验证 + 文档
   - JuiceFS 持久化路径清单的部署 manifest
   - GC stub session 后台扫描
   - tool authorization UX 与 WT 协议映射（依赖 Epic #6）
4. workspace root 的 `SESSION-HANDOFF-2026-04-30-sdk-migration.md` 删除（已被本 spec 取代）

### v0.2+（独立 ticket）

按 §3.1 表格归位。

### v0.1 内但不在本 ticket（协作面）

- Epic #5（E2E 加密）：本 ticket 提供 daemon ↔ WT 边界上的 canonical event schema，#5 在其上加密层
- Epic #6（App UI）：本 ticket 提供 daemon 推到 UI 的消息流定义，#6 决定渲染
- Epic #7（Tauri WT command）：本 ticket 与 #7 共享 WT 协议消息层，#7 负责 Tauri 侧 native 接入

---

## 10. Open questions（5 条设计问题）

### 10.1 用户已装的 hooks / MCP servers / settings.json 是否传给 cc 子进程？

- **复用**（继承用户的 `~/.claude/settings.json` 全部）：与用户 CLI 行为一致；但用户的 hook 副作用 + MCP server lifecycle 跟 tether daemon 交互复杂。
- **隔离**（tether 注入临时 settings 屏蔽用户的）：干净；但 tether 要重建用户期望的工具集，UX 容易割裂。
- **spike 推荐**：复用整体配置，tether 自己的 hook 严格 idempotent（C.4），不试图屏蔽用户配置。

### 10.2 工具授权 UX 怎么映射到 tether WT 协议？

- mid-stream cancel？batch approve？deny + ask-followup？
- **依赖 Epic #6 UI 设计**，本 spec 不定稿；但 daemon 端的 `control_request can_use_tool` 完整握手必须先实现（§4.5），UI 怎么用是上层的事。

### 10.3 凭据注入路径（ANTHROPIC_API_KEY / OAuth）

- per-pod k8s Secret + env var？
- 用户在 tether app 登录 → daemon 获取 token → 注入子进程 env？
- vault？
- **依赖 Epic #5 / 部署 ticket**，本 spec 不定稿；spike 阶段用本地 OAuth 即可。

### 10.4 cwd 策略二选一定稿

- **(a) fixed cwd（`$HOME`）**：实现简单，session 全部在一个 bucket，杂；用户裸 cc resume 在 $HOME 即可。
- **(c) `tether resume` wrapper**：屏蔽 cwd 概念；但用户得用 wrapper，不能裸 cc。
- **二选一在 spike 阶段定稿**，作为 §6 E.5 的具体实现选择。

### 10.5 一用户多 Session 模型

- 现状：handoff 和本 spec 反复说"一用户一容器"，但**没明说**一用户能否有多个**并发** Session（同账号手机+笔记本同时聊不同主题？同设备开多 chat tab？）。
- 模型选择：
  - **(i) 单 Session per daemon**——简单，但限制用户只能一个对话；不符合产品形态
  - **(ii) 多 Session per daemon**——单 daemon 同时拥有 N 个独立 cc 子进程，每个独立 session_id；OAuth/quota 在用户层共享
- **倾向 (ii)**——daemon 是 multiplexer。spike 验证 N=2 没问题即可；上限留给压测决定。
- 影响：`Session` 必须是 lib 实例（不是 singleton）；daemon 维护 `sessions map[string]*Session` 加 mutex 保护。
- spike 阶段验证：scenario 1 跑两个并发 Session（不同 user msg），互不干扰；各自 `Recover()` 不影响对方。

---

## 11. References

- Original handoff: `<workspace-root>/SESSION-HANDOFF-2026-04-30-sdk-migration.md`
- happy-cli 源码（read-only reference）: `github.com/slopus/happy-cli`
  - `src/claude/sdk/query.ts` — spawn + NDJSON 主循环 + control 协议
  - `src/claude/claudeLocal.ts` — fd 3 副信道（**本 spec 已弃**）
  - `src/claude/session.ts` — Session 结构（无对话存储）
  - `src/claude/loop.ts` — local/remote 双模式状态机（**tether 不需要**）
- Claude Code 官方文档:
  - Run Claude Code programmatically: <https://code.claude.com/docs/en/headless>
  - GH issue #24594 — stream-json docs gap: <https://github.com/anthropics/claude-code/issues/24594>
- 论证记录:
  - 7 站可行性论证（A: 接口稳定 / B: 控制协议 / C: kill+resume / D: 部署形态 / E: jsonl 复用 / F: 嵌 Node 重估 / G: backend 抽象）—— 2026-04-30 brainstorm session
  - cwd-gap 实测：从 `/tmp/claude-resume-probe` 起 session → 从 `/root` resume → `No conversation found`（确认 cwd-scoped 行为）
