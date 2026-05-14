# D-17a: AgentProvider — 多 provider 集成路径与踩坑表

**Date:** 2026-05-09
**Author:** wxk + Claude
**Status:** Proposed (in-flight，跟 D-22 / D-05a 一同等用户拍板新方向后正式纳入主 spec)
**Companion docs:**
- 主 spec D-17 `2026-04-26-tether-go-quic-design.md` §11.W
- D-22 `2026-05-09-d22-tygo-schema-sync.md`
- D-05a (pending)

---

## 0. 这份文档解决什么

主 spec D-17 锁定 v0.1 仅 cc 一个 provider，"加新 provider 在第二个 provider 实作时再稳定接口"。**没具体写未来怎么扩。**

cloudcli 实践里 4 个 provider（claude / cursor / codex / gemini）+ PoC step9 验证 opencode 也能接，**让我们提前看清未来扩展路径**：分两类，每类的踩坑明确，避免每加一个 provider 各自踩一遍。

D-17a 不改 D-17 的"v0.1 仅 cc 实现"承诺，**只把"未来怎么扩"写到能查得到的地方**。

## 1. 两类 provider — 集成机制对照

| Provider | 集成方式 | 多 turn 模型 | LOC | 关键 flag / API |
|---|---|---|---|---|
| **Claude (cc)** | stream-json **长跑** subprocess | 单进程多 prompt（input-format stream-json） | ~800-1500 (step8 已 PoC) | `--print --output-format stream-json --input-format stream-json --verbose --resume <sid>` |
| **Cursor** | spawn `cursor-agent` (per-message) | 每 prompt 重启 subprocess | ~300-400 | `--resume=<sid> -p <cmd> --output-format stream-json -f` |
| **Codex** | **Node SDK** `@openai/codex-sdk` | SDK AsyncIterator | ~400 | item events: `agent_message` / `reasoning` / `command_execution` / `file_change` |
| **Gemini** | spawn `gemini` (per-message) | 每 prompt 重启 subprocess | ~500 | `--prompt <cmd> --resume <cliSid>`，自定义 response handler |
| **Opencode** | spawn `opencode run` (per-message) | session 持久化在磁盘，每 turn 独立 subprocess | ~80（PoC step9 实测） | `run --format json -s <sid> <prompt>` |

**两类**：
- **类 A（CLI subprocess + JSON stream）**：claude / cursor / gemini / opencode。subprocess + 解析 newline-delimited JSON 事件。可纯 Go，不依赖 Node。
- **类 B（Node SDK）**：codex。需要 Node 运行时。

## 2. 子类 A 的两个变种

### A1. 长跑多 turn（claude 模式）

claude 的 `--input-format stream-json` 让单 process 处理多 prompt：

```
[once per session]   spawn claude --print --output-format stream-json
                                  --input-format stream-json --verbose
[per user message]   write JSON to stdin: {"type":"user","message":{"role":"user","content":"..."}}
[continuously]       parse line-delimited JSON events from stdout
                       — system/init, assistant, result, system/hook_*, rate_limit_event
```

**优点**：0 启动开销跨 prompt（PoC step8 实测 ~3s 启动一次，后续 prompt 即时）。
**约束**：需要 cc 2.x+。

### A2. 每 prompt spawn（opencode / cursor / gemini 模式）

每条用户消息独立 spawn：

```
[per user message]   spawn opencode run --format json -s <sid> "<prompt>"
[until exit]         parse line-delimited JSON events from stdout
                       — step_start, text, step_finish (opencode)
                       — claude-style stream-json (cursor)
                       — custom format (gemini)
```

**优点**：实现简单；每 prompt CLI 自己处理状态保存，daemon 无状态。
**代价**：每 prompt ~1-3s 启动开销（看 CLI 实现）。
**约束**：CLI 必须自管 session 持久化（opencode 验证过，session 状态磁盘留存，跨 subprocess 记得历史）。

## 3. tether 的 AgentProvider Go interface 草案

提议 `internal/agent/provider.go`：

```go
package agent

import "context"

// AgentProvider abstracts a CLI agent integration. Implementations cover
// the two variants from D-17a §2 (long-running A1 / per-prompt A2).
type AgentProvider interface {
    // Name is the canonical provider id ("claude", "opencode", ...).
    // Used in wire envelope `providerType` field (D-14).
    Name() string

    // Spawn starts the provider for a session. For A1 (long-running),
    // this spawns the process and keeps it alive. For A2 (per-prompt),
    // this is a no-op or just records the session id.
    //
    // Returns a Session handle bound to ctx; cancelling ctx cleans up.
    Spawn(ctx context.Context, sessionID string, opts SpawnOptions) (Session, error)
}

// Session is one ongoing user-agent conversation.
type Session interface {
    // SendPrompt feeds one user prompt. For A1, writes to stdin of the
    // long-running process. For A2, spawns a fresh subprocess.
    //
    // Events stream back via the channel returned by Events().
    SendPrompt(ctx context.Context, prompt string) error

    // Events returns the event channel for this session. Producer is
    // the provider impl; consumer is the daemon's chat goroutine.
    Events() <-chan Event

    // Interrupt cancels the in-flight prompt (if any). For A1, sends
    // a control message; for A2, kills the active subprocess.
    Interrupt(ctx context.Context) error

    // Close ends the session. For A1, closes stdin (process exits cleanly).
    // For A2, no-op since each prompt's subprocess already exited.
    Close() error
}

// Event is a normalized message after provider-specific event mapping.
// All providers map their native events to this shape.
type Event struct {
    Kind      EventKind       // assistant_text / tool_use / permission / result / system / error
    Text      string          // human-displayable text (for assistant_text / system)
    ToolUse   *ToolUseEvent   // populated when Kind == tool_use
    SessionID string
    Raw       json.RawMessage // original provider event for debugging
}
```

**形状由 cloudcli 验证过**——它在 JS 端用约定（不是真接口），但函数签名、生命周期、normalization 思路都是这个。tether 用 Go interface 把约定显式化。

## 4. v0.1 影响

**D-17a 不改 v0.1 范围。** 主 D-17 锁定的"仅 cc 实现"不变。新增的只是：

1. 把上述 interface 提前定义在 `internal/agent/provider.go`，**v0.1 ClaudeCodeProvider 是唯一实现**
2. 其他 provider（opencode / cursor / gemini / codex）在 `internal/agent/` 里**仅留 stub 文件 + 注释**说明扩展路径，无实现
3. 公开 surface 仍**不暴露 multi-provider 能力**（无 `tether agents` CLI、无 `--agent` flag）—— 跟 D-17 一致

**LOC 增加**：~30 LOC（interface 定义 + 4 个 stub 文件）。

## 5. 各 provider 的踩坑表（实施期参考）

### 5.1 Claude（v0.1 实现）

- ✅ stream-json 长跑模式：单 process 多 turn（step8 验证）
- ✅ session_id 跨 turn 稳定：从 `system/init` 事件提取
- ⚠️ `system/init` 每 turn 触发**一次**（不是每 process 一次）—— 不要把 init 当 session boundary
- ⚠️ `--dangerously-skip-permissions` 在 root 用户下 cc 拒绝，需要 `IS_SANDBOX=1` env 绕过（已实测）
- ⚠️ Hook events (`hook_started` / `hook_response`) 也走 stream-json，需要 `--include-hook-events` flag 才输出（默认开还是关待确认）

### 5.2 Cursor（未实现，cloudcli 经验）

- ⚠️ **Workspace trust 提示**：cursor-agent 第一次进未信任目录会卡住；需要正则匹配 4 个 pattern 自动加 `--trust` 重试
  - `/workspace trust required/i`
  - `/do you trust the contents of this directory/i`
  - `/working with untrusted contents/i`
  - `/pass --trust,\s*--yolo,\s*or -f/i`
- ⚠️ Per-prompt spawn 模型，每条消息 ~2-3s 启动开销
- ✅ 跟 claude 同款 `--output-format stream-json`，事件解析逻辑可复用

### 5.3 Gemini（未实现，cloudcli 经验）

- ⚠️ **CLI session ID ≠ tether session ID**：需要 `sessionManager` 类似的映射层（cloudcli 用 5KB JS 文件做这事）
- ⚠️ **图片上传需要 temp 文件**：gemini CLI 不支持 stdin 图片，必须写 temp 文件，把路径塞 prompt
- ⚠️ **不输出 stream-json**：自定义解析器（cloudcli 用 `gemini-response-handler.js`）
- ⚠️ 路径需 ASCII 净化：`replace(/[^\x20-\x7E]/g, '').trim()`（cloudcli 实测踩过）

### 5.4 Codex（v1.0+ 候选）

- ⚠️ **Node SDK only**：`@openai/codex-sdk` 是 npm 包，不能直接 Go 集成
- 三条入路：
  1. **Node sidecar process**：tether daemon spawn 一个小 Node 进程跑 SDK，stdin/stdout 自定义协议——破坏 single-binary 叙事
  2. **直接调 OpenAI API + 自实现工具运行时**：scope 巨大，重写一遍 cc/cursor/codex 干的事
  3. **等 codex CLI 出 stream-json 模式**：观察 OpenAI 是否后续加 `codex run --output-format json`
- v0.1 / v0.5 范围内**直接砍掉**——不值得。

### 5.5 Opencode（v1.0+ 候选，PoC step9 验证）

- ✅ `opencode run --format json -s <sid> <prompt>` per-prompt spawn 模式
- ✅ Session 跨 subprocess 持久化（disk-based）
- 事件类型：`step_start` / `text` / `step_finish`，结构 `{type, timestamp, sessionID, part}`
- `part.text` 是 assistant 输出
- ⚠️ **opencode 还自带 `serve`/`acp` 模式**——是个完整的 server 自身。tether 集成 opencode 也可考虑"tether 通过 HTTP/ACP 跟 opencode server 对接"，**比 spawn CLI 更现代**，但增加 HTTP client 依赖
- 实现简单度估算：spawn CLI ~80 LOC，HTTP/ACP ~150-200 LOC

## 6. 优先级建议

```
v0.1            v0.2           v0.5            v1.0+
─────────       ─────────      ─────────       ─────────
claude (cc)     opencode       cursor          codex
                                gemini          (Node sidecar
                                                or wait API)
```

**理由**：
- v0.1：只 cc，跟 D-17 一致
- v0.2：opencode 集成最简单（已 PoC，~80 LOC，纯 stream-json）
- v0.5：cursor + gemini 实施期多踩坑（workspace trust / sid mapping），值得排在 opencode 后面
- v1.0+：codex Node 依赖太重，等 OpenAI 出原生 CLI stream-json 模式或 v1.0 重新评估架构是否容忍 Node sidecar

## 7. 跟 D-22 / D-05a 的衔接

- **D-22（tygo schema sync）**：**`Event` / `EventKind` 是 daemon 内部翻译边界类型**（agent → daemon → wire envelope），定义在 `internal/agent/event.go`，**不进** `internal/wire/`，**不被 tygo 生成**。daemon 把 ProviderEvent 翻译成 `wire.Envelope` / `wire.FencedBlock` 才发给前端。前端只看到 `internal/wire/` 里 tygo 生成的 TS 类型，不感知 provider 来源（除了 `providerType` 标签用于显示）
- **D-05a（双通道 chat + shell）**：D-17a 的 AgentProvider 仅管 chat 通道。Shell 通道通用 PTY，不需要 per-provider 实现——任何 CLI 都能 PTY-attach（claude / cursor / gemini / opencode 都支持 raw TTY 模式）

## 8. Acceptance criteria

- [ ] `internal/agent/provider.go` 定义 AgentProvider + Session interface
- [ ] `internal/agent/claude_provider.go` v0.1 实现（长跑 stream-json，覆盖 5.1 所有 ⚠️）
- [ ] `internal/agent/{opencode,cursor,gemini,codex}_provider.go` 各 1 个 stub 文件，注释指向 §5.X 踩坑表
- [ ] PoC step8 + step9 的产出（事件类型清单 / session_id 行为 / 长跑 vs per-msg cost 对比）写进各 provider 文件 godoc 顶部
- [ ] `internal/agent/event.go` 定义 `Event` / `EventKind` / `ToolUseEvent`（**不被 tygo 生成**——这是 daemon 内部边界类型，不跨 daemon ↔ browser）
- [ ] daemon 翻译层把 `Event` 转 `wire.Envelope` / `wire.FencedBlock`（生成 TS 类型）后发前端
- [ ] 主 spec D-17 加一行链接到 D-17a，但不修改 D-17 锁定的"v0.1 仅 cc"承诺

## 9. Addendum — opencode 实施漂移到 A1 (2026-05-14)

§3.A2 把 opencode 归到「每 prompt spawn」模式，理由是 `opencode run --format
json` 足以覆盖功能。但 v2 实施期发现：

- `opencode run` 的 JSON 输出 **不包含 token-level 文本流**——只在 `message.part`
  完成后整块给出文本。这跟 cc 用 `stream-json` 流式输出体验完全错位。
- opencode 自带的 `serve` 子命令开了一个本地 HTTP 服务，`/global/event` SSE
  endpoint **会**广播 `message.part.delta`（token 级文本增量），跟 cc 的
  `assistant.content_block_delta` 体验匹配。

所以 `internal/agent/opencode_provider.go` 实际走的是 §3.A1（长跑 serve +
SSE）+ §3.A2（per-prompt `opencode run --attach <url>`）的混合：

```
[once per session]   spawn opencode serve --port <free>
                     subscribe SSE: GET /global/event
                     → message.part.delta → EventText
                     → session.created    → EventInit

[per user prompt]    spawn opencode run --attach <baseURL> "<prompt>"
                     → goroutine 解析 stdout 捕获 session ID
                     → cmd.Wait() 退出后 EventResult
```

**取舍**：
- ✅ token-level streaming 跟 cc 对齐（dogfood: 20 chunks / 3 段文本，2026-05-14）
- ✅ session 由 opencode serve 自管，跨 turn 自带 history
- ⚠️ 启动多了 `serve` subprocess + 任意 free port — TOCTOU 窗口已通过
  `serve.Wait()` 监控收紧（详见 opencode_provider.go::waitReady）
- ⚠️ §3.A2 表里 opencode 行需要在下一次 spec revision 时勘误为 A1+A2 混合

**为什么没在表里直接改**：D-17a v1 表述已固化在 v0.2.0 dogfood 备忘，跨多
个 task 引用；改表会破坏对外文档稳定性，本 addendum 段在 v0.4.x cycle
做一次合并。
